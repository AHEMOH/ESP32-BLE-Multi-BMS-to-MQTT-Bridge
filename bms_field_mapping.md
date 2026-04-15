# BLE BMS Field Mapping (ESPHome MQTT)

Stand: 2026-04-10

Dieses Dokument beschreibt den aktuellen, praxiserprobten Feldstand fuer [bms_bt_mqtt.yaml](bms_bt_mqtt.yaml).
Es basiert auf realen LiTime- und PowerQueen-Frames, App-Vergleichswerten und laufender Validierung im Produktivbetrieb.

## 1) Telegramm-Basis

- Frame-Laenge: `105` Byte
- Checksumme: `sum(frame[0..103]) & 0xFF == frame[104]`
- Endianness: Little Endian fuer numerische Felder

## 2) Kern-Mapping (roh)

| Feld | Offset | Typ | Einheit | Status |
|---|---:|---|---|---|
| pack_voltage_mv_raw | 8 | u32_le | mV | bestaetigt |
| pack_voltage_mv_alt | 12 | u32_le | mV | offen/variant |
| cell1_mv | 16 | u16_le | mV | bestaetigt |
| cell2_mv | 18 | u16_le | mV | bestaetigt |
| cell3_mv | 20 | u16_le | mV | bestaetigt |
| cell4_mv | 22 | u16_le | mV | bestaetigt |
| current_raw | 48 | s32_le | mA | weitgehend bestaetigt |
| temp_cell_c | 52 | u16_le | degC | bestaetigt |
| temp_mosfet_c | 54 | u16_le | degC | bestaetigt |
| remain_ah_raw | 62 | u16_le | 0.01 Ah | weitgehend bestaetigt |
| factory_ah_raw | 64 | u16_le | 0.01 Ah | weitgehend bestaetigt |
| status_flags | 68 | u32_le | bitfield | bestaetigt |
| protect_state_raw | 76 | u32_le -> u8 | code | vorlaeufig |
| discharge_switch_legacy | 78 | u32_le -> u8 | code | nicht mehr genutzt |
| equil_state | 84 | u32_le -> u8 | code | teil-bestaetigt |
| operating_mode_raw | 88 | u16_le -> u8 | code | vorlaeufig |
| soc_pct | 90 | u16_le -> u8 | % | bestaetigt |
| soh_pct | 92 | u16_le -> u8 | % | bestaetigt |
| cycle_count | 96 | u32_le | count | bestaetigt |
| discharged_ah_total | 100 | u32_le | count/raw Ah | bestaetigt |

## 3) Abgeleitete Felder in MQTT

### Pro Batterie (`ble_battery/<battery_name>/...`)

- `voltage_mv`
- `pack_voltage_mv`
- `current_ma`
- `power_w`
- `remain_ah`
- `factory_ah`
- `cell_temp`
- `mosfet_temp`
- `soc`
- `soh`
- `cell1_mv..cell4_mv`
- `cell_delta_mv`
- `cell_delta_status`
- `equil_state`
- `equil_cell` (heuristisch aus `equil_state`-Bitmaske: `1->cell1`, `2->cell2`, `4->cell3`, `8->cell4`; bei mehreren Bits als CSV, z. B. `cell1,cell3`)
- `protect_state`
- `discharge_switch` (`ON`/`OFF`)
- `operating_mode_raw`
- `operating_mode`
- `cycle_count`
- `discharge_ah_count`
- `voltage_context`

### Pack/System (`ble_battery/<battery_system_name>/...`)

- `pack_voltage_mv`
- `pack_current_ma`
- `pack_capacity_ah`
- `pack_remain_ah`
- `pack_soc`
- `pack_power_w`
- `series_group_voltage_delta_mv`
- `slot_voltage_delta_mv`
- `pack_cell_delta_max_mv`
- `pack_cell_balancer_status`
- `pack_series_balancer_status`
- `pack_cycle_count_max`
- `pack_discharged_ah_total`
- `series_groups`
- `active_slots`
- `expected_slots`
- `topology_reporting` (z. B. `2/2`)
- `topology_complete` (`true`/`false`)

## 4) Schalter- und Statuslogik

- `status_flags@68` Bit `0x80` wird als Entlade-Sperre interpretiert.
- Daraus wird `discharge_switch` abgeleitet:
  - `OFF`, wenn Bit `0x80` gesetzt ist
  - `ON`, wenn Bit `0x80` nicht gesetzt ist

Wichtig:
- Das ist explizit ein Entladepfad-Signal.
- Laden kann weiterhin moeglich sein, auch wenn `discharge_switch=OFF`.

## 5) Balancing-Interpretation

### Zell-Delta (innerhalb einer Batterie)

Aktuelle Schwellwerte:
- `good <= 10 mV`
- `light <= 20 mV`
- `warning <= 40 mV`
- `critical > 40 mV`

### Seriengruppen-Delta (zwischen 12V-Packs im System)

Aktuelle Schwellwerte:
- `good <= 20 mV`
- `light <= 50 mV`
- `warning <= 120 mV`
- `critical > 120 mV`

## 6) Beobachtete `equil_state`-Hinweise (Hypothese)

Fuer Batterie `L-12100BNNA70-XXXXXX`:
- `equil_state = 1` -> Vermutung: Cell 1 wird aktiv gebalanced
- `equil_state = 2` -> Vermutung: Cell 2 wird aktiv gebalanced
- `equil_state = 4` -> Vermutung: Cell 3 wird aktiv gebalanced
- `equil_state = 8` -> Vermutung: Cell 4 wird aktiv gebalanced

Status: noch nicht final bestaetigt, aber reproduzierbar beobachtet.

## 7) Fallback-Regeln im Parser

- `cycle_count`: primaer `u32@96`, fallback `u16@66` wenn primaer unplausibel
- `discharge_ah_count`: primaer `u32@100`, fallback `u32@68` wenn primaer unplausibel

## 8) Offene Punkte fuer weiteres Reverse Engineering

- Exakte Semantik von `u32@12`
- Vollstaendige Bedeutung der Flags in Bereich `62..75`
- Vollstaendige Semantik von `operating_mode_raw@88`
- Finale Dekodierung aller `equil_state`-Codes

Neuer PQ-Frame (Charge, ca. +20 A):

- Checksumme korrekt: `A7`
- `u16_le@90`: `84 -> 85 -> 86 -> 87` (Idle -> -8 A -> +15 A -> +20 A)
- `u16_le@62`: `8475 -> 8575 -> 8742 -> 8802` (remainAh-Kandidat weiter bestaetigt)
- `u16_le@64`: weiterhin konstant `10058` (factoryAh-Kandidat weiter bestaetigt)

Lastabhaengiger Kandidatbereich `48..49` (u16_le):
- Idle: `0`
- Entladen ca. -8 A: `28467`
- Laden ca. +15 A: `14027`
- Laden ca. +20 A: `18840`

Interpretation:
- Der Bereich `48..49` traegt klar Lastinformation, aber die reine lineare Skalierung auf A ist noch nicht direkt ersichtlich.
- Moeglich sind Bias-/Bitfeld-/Transform-Varianten oder die Abbildung auf Leistung statt Strom.

Naechster Verifikationsschritt:
- 3 bis 5 weitere Punkte nur auf PQ sammeln (mit stabilen Lasten und genauer App-Stromangabe),
  dann gezielt folgende Modelle testen:
  - signed/unsigned u16 LE direkt
  - Offset-Bias (`raw - c`)
  - invertierte Darstellung (`65535 - raw`)
  - Korrelation zu Leistung `P = U * I` statt direkt zu I

### 8.10 Zusaetzlicher PQ-Frame: Ladegeraet getrennt

Beobachtungen nach Trennen des Ladegeraets:

- Checksumme weiterhin korrekt: `E0`
- `u16_le@48` faellt auf `0` zurueck (von `18840` bei +20 A)
- `u16_le@62` steigt weiter: `8802 -> 8867` (remainAh-Kandidat konsistent)
- `u16_le@64` bleibt `10058` (factoryAh-Kandidat stabil)
- `u16_le@90`: `87 -> 88` (SOC steigt weiter)
- `u16_le@68` bleibt `0` (Discharge-OFF-Bit nicht gesetzt)

Interpretation:
- Das lastabhaengige Feld bei `48..49` reagiert nicht nur auf Laststaerke,
  sondern faellt im no-load/charger-off Zustand wieder auf 0.
- Das stuetzt die Einstufung als aktueller Hauptkandidat fuer Strom-/Leistungsnaehe.

### 8.11 Zweiter PQ-Frame mit weiter getrenntem Ladegeraet

Der Folgesample im gleichen Zustand (Ladegeraet weiterhin getrennt) bestaetigt die No-Load-Signatur:

- Checksumme korrekt (`6C`)
- `u16_le@48` bleibt `0` (stabil im no-load Zustand)
- `u16_le@62` bleibt `8867`
- `u16_le@64` bleibt `10058`
- `u16_le@90` bleibt `88`
- `u16_le@68` bleibt `0`
- `u32_le@96` bleibt `1`, `u32_le@100` bleibt `191`

Nur erwartbare Relaxation/Drift beobachtet:
- Pack-/Zellspannungen sinken leicht (`@8`, `@12`, `@16..22`)

Schlussfolgerung:
- Die Felder `48`, `62`, `64`, `90`, `96`, `100` verhalten sich in dieser Sequenz sehr stabil.
- Das spricht gegen Zufall und staerkt die aktuelle Mapping-Richtung deutlich.

### 8.12 PQ no-load mit `dischargeSwitch = OFF`

App-Referenz:
- SOC 88%, Temperatur 26 C, Leistung 0 W, Strom 0 A, Spannung 13.4 V, Cycle 1

Frame-Beobachtung:
- Checksumme korrekt (`E2`)
- `u16_le@68`: `0 -> 128` (Bit `0x80` gesetzt, konsistent mit Discharge OFF)
- `u16_le@48`: bleibt `0` (passt zu no-load, 0 W / 0 A)
- `u16_le@52/@54`: `26/26` (passt zur App-Temperatur)
- `u16_le@90`: `88` (SOC passt)
- `u32_le@96`: `1`, `u32_le@100`: `191` (zaehlerstabil)

Auffaellig, aber bereits bekannt:
- `u32_le@8` faellt bei gesetztem OFF-Bit auf einen unplausiblen Wert (`7814` mV),
  waehrend App-Spannung bei ca. `13.4` V liegt.
- Damit wird die Plausibilitaetsregel fuer Spannung bei OFF/Standby weiter bestaetigt.

Aktualisierte Interpretation:
- Der niedrige OFF-Wert ist wahrscheinlich eine echte Pol-/Klemmenspannung
  am Ausgang bei offenem Switch (nicht zwingend ein Ausreisser).

### 8.13 LiTime-Frame mit ca. 8 A Belastung

Neuer LiTime-Frame unter Last zeigt ein starkes Stromfeld-Signal:

- Checksumme korrekt (`3F`)
- `u32_le@48` = `0xFFFFEA5B`, also `s32_le@48 = -5541`
- Im LiTime-Idle war `s32_le@48 = 0`
- Unter Last aendern sich genau Bytes `48..51` (`00 00 00 00 -> 5B EA FF FF`)

Interpretation:
- `s32_le@48` ist ein starker Kandidat fuer Strom in mA (oder eng verwandte Lastgroesse).
- Vorzeichen passt zur Entlade-/Belastungsrichtung (negativ unter Last).
- Skalierung ist noch vorlaeufig: `-5541` entspricht groessenordnungsmassig mehreren Ampere,
  aber benoetigt weitere LiTime-Referenzpunkte fuer finale Kalibrierung.

Nebenbefund:
- Bei diesem LiTime-Lastframe fiel `u16_le@68` auf `8` (vorher ON-idle `72`, OFF-idle `200`).
- Das bestaetigt: Offset 68 enthaelt mehrere zustandsabhaengige Flags, nicht nur das OFF-Bit.

### 8.14 LiTime-Frame mit App-Referenz: 13.5 V, -7.8 A, -109 W, 99 Ah

Neuer LiTime-Frame mit klarer App-Referenz bestaetigt die Strom-Hypothese sehr stark:

- Checksumme korrekt (`1D`)
- `s32_le@48 = -8074`
- App-Strom: `-7.8 A`

Starke Korrelation:
- Wenn `current_a = s32_le@48 / 1000.0`, ergibt sich `-8.074 A`
- Das liegt sehr nahe am App-Wert `-7.8 A`

Leistungs-Check:
- `u32_le@8 = 13414 mV` -> `13.414 V`
- `13.414 V * (-8.074 A) = -108.3 W`
- App-Leistung: `-109 W` (sehr gute Uebereinstimmung)

Weitere passende Felder:
- `u16_le@90 = 99` (SOC 99)
- `u16_le@52/@54 = 27/27` (Temperatur 27)
- `u32_le@96 = 1` (Cycle 1)

Balancing-Hinweis:
- In diesem Frame ist `u16_le@88 = 2` (wie bei anderen LiTime-Lastframes).
- Das ist ein plausibler Balancing-Kandidat, aber noch nicht final bestaetigt.

Kapazitaetsfeld bei LiTime:
- `u16_le@62 = 10046`, `u16_le@64 = 10089`
- Das spricht fuer eine LiTime-nahe remain/factory-Logik mit variantenspezifischer Skalierung/Glattung.

### 8.15 LiTime-Frame mit ausgeschaltetem Entladeschalter

Neuer LiTime-OFF-Frame bestaetigt die OFF-Signatur:

- Checksumme korrekt (`15`)
- `u16_le@68 = 128` (`0x80` gesetzt)
- `s32_le@48 = 0` (kein Laststrom, konsistent zu OFF/no-load)
- `u16_le@90 = 99`, `u16_le@52/@54 = 27/27`, `u32_le@96 = 1` (stabil)

Spannungsmodus bei OFF:
- `u32_le@8 = 2731` mV, deutlich niedriger als Packspannung im aktiven Zustand
- Das passt zur zuvor diskutierten OFF-Interpretation (Pol-/Klemmenspannung bei offenem Schalter)

LiTime-Kapazitaetsfelder:
- `u16_le@62 = 10027`, `u16_le@64 = 10089`
- Damit bleibt `@64` als factoryAh-Kandidat stabil,
  waehrend `@62` dynamisch auf Last/Zustand reagiert (remainAh-Kandidat).

### 8.16 Zweiter LiTime-Frame bei unveraendertem OFF-Zustand

Folgeframe ohne Zustandsaenderung (Schalter bleibt OFF) bestaetigt die Stabilitaet:

- Checksumme korrekt (`23`)
- `u16_le@68 = 128` bleibt gesetzt
- `s32_le@48 = 0` bleibt unveraendert
- `u16_le@62 = 10027`, `u16_le@64 = 10089` bleiben unveraendert
- `u16_le@90 = 99`, `u16_le@92 = 100`, `u32_le@96 = 1`, `u32_le@100 = 104` bleiben unveraendert

Nur leichte Drift in Spannungsfeldern:
- `u32_le@12: 13467 -> 13474`
- Zellwerte `u16_le@16/18/20/22`: leichter Anstieg um wenige mV

Schlussfolgerung:
- OFF-Signatur ist bei LiTime nun ueber mehrere Folgeframes reproduzierbar und stabil.

### 8.17 LiTime wieder ON mit ca. 8 A Last + frischer PQ-OFF-Frame

LiTime (Entladeschalter wieder AN, Last aktiv):

- Checksumme korrekt (`DD`)
- `u16_le@68`: `128 -> 0` (OFF-Bit zurueckgenommen)
- `s32_le@48` springt wieder auf Lastwert: `0 -> -8074`
- `u16_le@88`: `0 -> 2` (wie bei anderen LiTime-Lastframes)
- `u32_le@100`: `104 -> 105` (Entlade-Ah-Zaehler steigt)

Interpretation:
- ON/OFF-Status und Lastfeld sind fuer LiTime konsistent reproduzierbar.
- `s32_le@48 / 1000` bleibt als Strom-Hauptmapping stabil.

Frischer PowerQueen-Frame (Schalter weiterhin OFF):

- Checksumme korrekt (`BC`)
- OFF-Signatur unveraendert stabil:
  - `u16_le@68 = 128`
  - `s32_le@48 = 0`

## 9) Command-Check fuer 0x10 / 0x16 / 0x41 / 0x43

Status per 2026-04-10:

| CMD | Request-Frame (8 Byte) | Zweck | Extern bestaetigt | In diesem YAML aktiv |
|---|---|---|---|---|
| `0x10` | `00 00 04 01 10 55 AA 14` | Serial Number | ja | nein |
| `0x16` | `00 00 04 01 16 55 AA 1A` | Firmware Version | ja | nein |
| `0x41` | `00 00 04 01 41 55 AA 45` | SOH/SOC (separat) | ja | nein |
| `0x43` | `00 00 04 01 43 55 AA 47` | Nominal Capacity | ja | nein |

Hinweis zur aktuellen Implementierung:

- In `bms_bt_mqtt.yaml` wird derzeit nur `0x13` gesendet:
  - `00 00 04 01 13 55 AA 17`
- Die vier Kommandos sind dokumentiert und in externen Projekten belegt,
  aber in diesem Projekt noch nicht aktiv gepollt.

## 10) Auswertung der zwei bereitgestellten Beispiel-Frames

Eingangsdaten:

- `PQ`: `000065019355AA008B370000343600008F0D940D880D890D000000000000000000000000000000000000000000000000000000001A001A000000000000004A274A2700004800000004000000000000000000000000000000000064006400000001000000BF00000076`
- `LI`: `000065019355AA00D236000026360000860D9C0D700D940D000000000000000000000000000000000000000000000000000000001A00190000000000000069276927000048000000040000000000000000000000040000000000640064000000010000006B0000008D`

Validierung:

- Beide Frames haben exakt `105` Byte.
- Beide Checksummen sind korrekt (`PQ: 0x76`, `LI: 0x8D`).
- Marker `byte[2] == 0x65` ist bei beiden gesetzt (Statusframe).

Dekodierte Kerndaten (nach aktuellem Mapping):

| Feld | PQ | LI | Kommentar |
|---|---:|---:|---|
| `u32@8` pack_voltage_mv_raw | `14219` | `14034` | plausibel |
| `u32@12` pack_voltage_mv_alt | `13876` | `13862` | variant/offen |
| `cell1..4 mV` | `3471/3476/3464/3465` | `3462/3484/3440/3476` | plausibel |
| `s32@48` current_raw | `0` | `0` | no-load |
| `u16@52/@54` temp | `26/26` | `26/25` | plausibel |
| `u16@62` remain_ah_raw | `10058` | `10089` | konsistent |
| `u16@64` factory_ah_raw | `10058` | `10089` | konsistent |
| `u32@68` status_flags | `0x00000048` | `0x00000048` | Bit `0x80` nicht gesetzt |
| `discharge_switch` (abgeleitet) | `ON` | `ON` | aus `status_flags@68` |
| `u32@76` protect_state_raw | `0` | `0` | unauffaellig |
| `u32@80` failure_state_raw | `0` | `0` | unauffaellig |
| `u32@84` equil_state | `0` | `4` | LI zeigt aktive/nonzero Signatur |
| `u16@88` operating_mode_raw | `0` | `0` | Idle/Standby-typisch |
| `u16@90` soc_pct | `100` | `100` | konsistent |
| `u16@92` soh_pct | `100` | `100` | konsistent |
| `u32@96` cycle_count | `1` | `1` | konsistent |
| `u32@100` discharged_ah_total | `191` | `107` | plausibel als Zaehler |

Fazit zu den zwei Frames:

- Sie bestaetigen das aktuelle `0x13`-Statusmapping sehr gut.
- Sie liefern keine direkte Antwort auf die Payloads von `0x10/0x16/0x41/0x43`,
  da beide Frames bereits `0x65`-Statusantworten sind.
- Fuer die vier Zusatzkommandos braucht es gezielte Request/Response-Captures je CMD.

## 11) Balancer-Verhalten in der Praxis (Zeitversatz beachten)

Praxisbefund (Live-Grafik):

- Wenn eine Zelle als zu hoch erkannt wird, wird sie aktiv heruntergezogen (Bleeding),
  teils sogar kurzfristig unter die anderen Zellen.

Folge fuer Frame-Interpretation:

- Ein einzelner Spaet-Frame kann deshalb ein anderes "hoechste Zelle"-Bild zeigen
  als die Ausloesesituation des Balancer-Flags.
- Das erklaert beobachtete Faelle wie `equil_state = 8`, obwohl im betrachteten
  Einzel-Frame nicht mehr Cell 4 die hoechste ist.

Aktueller Schluss:

- Die Bitmasken-Hypothese fuer `equil_state@84` bleibt plausibel.
- Zur belastbaren Verifikation sind zeitlich zusammenhaengende Frame-Sequenzen
  (nicht nur Einzel-Frames) entscheidend.

### 11.1 LI-Folgeframe: Zellvergleich mit deutlichem Drop auf Cell 4

Verglichene Frames (gleiche Batterie, kurzer zeitlicher Abstand):

- Alt: `000065019355AA005B3700004A360000AF0D7C0D830D9C0D...0000005E`
- Neu: `000065019355AA00433700003F360000AE0D7B0D820D940D...00000030`

Beide Frames:

- Laenge `105` Byte
- Checksumme korrekt
- `equil_state@84 = 8`

Zellspannungs-Deltas (neu minus alt):

- `cell1`: `3503 -> 3502` mV (`-1` mV)
- `cell2`: `3452 -> 3451` mV (`-1` mV)
- `cell3`: `3459 -> 3458` mV (`-1` mV)
- `cell4`: `3484 -> 3476` mV (`-8` mV)

Interpretation:

- In dieser Sequenz faellt Cell 4 deutlich staerker als die anderen Zellen.
- Das stuetzt die Zuordnung `equil_state = 8` zu einer aktiven Balancing-Aktivitaet auf Cell 4.

Praktische Empfehlung fuer Anzeige `equil_cell`:

- Wir koennen die Anzeige jetzt sinnvoll erweitern auf eine vorlaeufige Bitmaske:
  - `1 -> cell1`
  - `2 -> cell2`
  - `4 -> cell3`
  - `8 -> cell4`
- Kennzeichnung in UI weiterhin als "vorlaeufig/heuristisch", bis die Zuordnung ueber mehr Zeitreihen final bestaetigt ist.

Schlussfolgerung:
- Beide Varianten zeigen nun mehrfach reproduzierbare Zustandsmuster fuer OFF/ON und Last/no-load.

### 8.18 Experiment: PQ am Ladegeraet, App meldet OFF

Scenario laut App:
- Charging, ca. 19 A, ca. 250 W, Temperatur 26 C
- Schalter in App weiterhin OFF

Frame-Befund:
- Checksumme korrekt (`98`)
- `u16_le@68` ist `0` (nicht `128`)
- `u16_le@48 = 19094` (deutlich lastaktiv, zuvor OFF/no-load = `0`)
- `u16_le@88 = 1` (zuvor OFF/no-load meist `0`)
- `u16_le@62 = 8886`, `u16_le@64 = 10058`, `u16_le@90 = 88` (konsistent)

Interpretation:
- Das Protokollsignal bei `@68` zeigt in diesem Moment **nicht** OFF, obwohl die App OFF anzeigt.
- Moegliche Ursachen:
  1. Status-Latenz zwischen App-UI und BMS-Frame
  2. Getrennte Charge-/Discharge-Switch-Modelle in der App
  3. Bitbelegung bei aktivem Laden weicht vom no-load OFF-Fall ab

Praktische Parser-Regel (aktualisiert):
- `@68 & 0x80` weiterhin als starkes OFF-Indiz nutzen,
  aber bei gleichzeitig aktivem Lastfeld (`@48 != 0`) als "state conflict" markieren und debug-loggen.

Empfohlene Benennung im Code:
- `discharge_stop` oder `discharge_disabled`
- nicht generisch `battery_disconnected`

### 8.19 PQ wieder ohne Ladegeraet, ohne App-Aenderung

Neuer PQ-Frame nach dem Experiment (laut Beschreibung: Ladegeraet getrennt, App unveraendert):

- Checksumme korrekt (`37`)
- `u16_le@68 = 128` (OFF-Bit wieder gesetzt)
- `u16_le@48 = 0` (kein Lastfeld aktiv)
- `u16_le@88 = 0` (zurueck auf no-load Muster)
- `u16_le@62 = 8989`, `u16_le@64 = 10058`
- `u16_le@90 = 89`, `u16_le@92 = 100`, `u32_le@96 = 1`, `u32_le@100 = 191`

Delta gegen letzten OFF-Frame (`BC`):
- Hauptaenderungen in Spannungsfeldern `@8/@12/@16..22`
- `u16_le@62`: `8867 -> 8989`
- `u16_le@90`: `88 -> 89`

Interpretation:
- Das Protokoll faellt nach dem Lade-Experiment wieder konsistent in den OFF/no-load Zustand zurueck.
- Der vorherige Konfliktfall (App OFF, aber `@68=0` und Last aktiv) war damit sehr wahrscheinlich ein transientes oder zustandsabhaengiges Spezialfenster.

### 8.20 Neue Vermutung zu `equil_state` (Stand 2026-04-10)

Beobachtung aus MQTT (Battery `L-12100BNNA70-XXXXXX`):

- `equil_state = 2` -> Vermutung: Cell 1 wird aktiv entladen/gedroppt (Balancing)
- `equil_state = 4` -> Vermutung: Cell 3 wird aktiv entladen/gedroppt (Balancing)

Wichtig:
- Diese Zuordnung ist aktuell eine Arbeits-Hypothese, noch keine finale Bestaetigung.
- Sinnvoller Naechsttest: mehrere Frames mit gleichzeitiger Zellspannungsdifferenz loggen und pruefen,
  ob der gesetzte `equil_state`-Wert immer zur jeweils hoechsten Zelle passt.

Zusatzhinweis zu `dischargeSwitchState`:
- Das Legacy-Rohfeld (historisch aus anderem Offset) zeigte in der Praxis oft `0` und war nicht stabil interpretierbar.
- Fuer die Laufzeitlogik ist das Flag `discharge_off` aus `status_flags@68 & 0x80` robuster.

## 9) Was noch zu testen ist (Prioritaet)

### 9.1 Hohe Prioritaet

- PQ Strom-Skalierung final bestaetigen:
  - Mehrere stabile Punkte mit bekannter App-Stromangabe sammeln (z. B. -20, -10, -5, +5, +10, +20 A)
  - Gegen Kandidatfeld `u16_le@48` fitten (direkt, mit Bias, invertiert, ggf. Leistungskorrelation)
- Konfliktfall App OFF vs Frame ON absichern:
  - Mehrfach den Uebergang "Ladegeraet an/aus bei OFF in App" loggen
  - Zeitstempel und mehrere Folgeframes je Zustand erfassen
  - Pruefen, ob `@68` verzoegert kippt oder ein separates Charge-Switch-Modell existiert

### 9.2 Mittlere Prioritaet

- LiTime Balancing-Indikator verifizieren:
  - App Balancing AN/AUS gezielt schalten
  - Korrelation mit `u16_le@88` pruefen (aktuell plausibler Kandidat)
- LiTime remainAh-Skalierung finalisieren:
  - Entwicklung von `u16_le@62` ueber laengere Lastphasen gegen App-Kapazitaet vergleichen

### 9.3 Niedrige Prioritaet

- Feld `u32_le@12` semantisch klaeren (zweiter Spannungs-/Filterwert)
- Restflags im Bereich `62..75` und `84` feiner aufloesen

## 10) Visuelle Frame-Trennung (How-To)

Ziel:
- Einen 105-Byte-Frame schnell in logisch zusammenhaengende Bloecke teilen
- Beim manuellen Reverse Engineering sofort sehen, welcher Bereich zu welcher Bedeutung passt

### 10.1 Byte-Layout auf einen Blick

Format: `Offset..Offset (Laenge) = Bedeutung`

- `0..7 (8)` = Header/Framekennung
- `8..11 (4)` = pack_voltage_mode_a (zustandsabhaengig)
- `12..15 (4)` = pack_voltage_mode_b / sekund. Spannungswert
- `16..23 (8)` = cell1..cell4 (je `u16_le`, mV)
- `24..47 (24)` = aktuell meist 0 / reserviert
- `48..51 (4)` = Strom-/Last-Rohwert (LiTime stark: `s32_le`, mA)
- `52..53 (2)` = temp_1
- `54..55 (2)` = temp_2
- `56..61 (6)` = reserviert / variantenabhaengig
- `62..63 (2)` = remainAh-Kandidat (PQ/LI dynamisch)
- `64..65 (2)` = factoryAh-Kandidat (meist stabil)
- `66..67 (2)` = reserviert/legacy
- `68..71 (4)` = Statusflags (`0x80` = Entladestopp)
- `72..87 (16)` = variantenabhaengige Status-/Reserved-Bits
- `88..89 (2)` = Zusatzstatus (u. a. Last/Balancing-Kandidat)
- `90..91 (2)` = SOC
- `92..93 (2)` = SOH
- `94..95 (2)` = reserviert
- `96..99 (4)` = cycle_count
- `100..103 (4)` = discharged_ah_total
- `104 (1)` = checksum (`sum(bytes[0..103]) & 0xFF`)

### 10.2 Manuelle Markierung eines Rohframes

Vorgehen beim Lesen eines Hex-Strings:

1. In 2er-Hexpaare splitten (1 Paar = 1 Byte)
2. Byte-Index links mitschreiben (0, 1, 2, ...)
3. Die Offsets aus 10.1 farblich markieren, z. B.:
   - Blau: Spannungen (`8..23`)
   - Gruen: Strom/Last (`48..51`)
   - Orange: Temperaturen (`52..55`)
   - Gelb: Kapazitaet (`62..65`)
   - Rot: Flags (`68..71`)
   - Tuerkis: SOC/SOH/Zaehler (`90..103`)
4. Checksumme zuletzt gegen Byte 104 pruefen

### 10.3 Minimal-Template fuer schnelle Sichtpruefung

Frame ID: ...
State: ...

Header      [0..7]    :
Pack A      [8..11]   :
Pack B      [12..15]  :
Cells       [16..23]  : c1=..., c2=..., c3=..., c4=...
CurrentRaw  [48..51]  :
Temp1/Temp2 [52..55]  :
Remain/Fact [62..65]  :
Flags       [68..71]  :
Aux         [88..89]  :
SOC/SOH     [90..93]  :
Cycles      [96..99]  :
DisAh       [100..103]:
Checksum    [104]     :

Hinweis:
- Bei `Flags@68` immer Bit `0x80` separat notieren (Entladestopp).
- Bei OFF-Zustaenden Spannungsmodus (`@8`) als zustandsabhaengig behandeln.

## 11) LiTime-Balancer-Hypothese (zellselektiv)

Beobachtung aus der App:
- Beim Balancing scheint nicht nur AN/AUS zu gelten,
  sondern es wird jeweils eine konkrete Zielzelle beeinflusst.
- Die zuvor hoechste Zellspannung sinkt in der Folge,
  bei naechstem Event wird ggf. eine andere Zelle aktiv bearbeitet.

Arbeits-Hypothese:
- LiTime kodiert den aktiven Balancer-Zustand als Zellmaske/Index
  (nicht nur als booleschen Status).
- Kandidatenbereich bleibt primaer `@88..89` (und sekund. naheliegende Statusbereiche).

Update aus neuer Beobachtung:
- Der Wert `4` war nur zeitweise aktiv (wahrend Balancer-/Warnphase) und steht spaeter wieder auf `0`.
- Das spricht eher fuer ein phasenabhaengiges Statusflag (z. B. top-balance/over-voltage-nah)
  als fuer eine feste Zellindex-Bitmaske.

Pruefplan zur Verifikation:

1. Pro Balancer-Event mindestens 3 Frames loggen:
   - kurz vor Aktivierung
   - waehrend aktiver Entladung der Zielzelle
   - kurz nach Ende/Umschaltung
2. Parallel App-Werte sichern:
   - Zellspannungen pro Zelle
   - Anzeige, welche Zelle balanciert wird (falls sichtbar)
3. Erwartung fuer echtes Zellbitfeld:
   - Wert in Kandidatfeld springt synchron zum Event
   - Wenn Zielzelle wechselt, aendert sich Bit/Index reproduzierbar
4. Mapping-Ansatz:
   - Falls genau eine Zelle aktiv: Indexmapping (z. B. 1..4)
   - Falls mehrere Zellen aktiv: Bitmaskenmapping

Hinweis fuer Implementierung:
- Bis zur finalen Verifikation Feld als "balancer_raw" belassen,
  aber Debug-Logging mit Zellspannungs-Korrelation aktivieren.

## 12) LiTime Lade-Start Sequenz (neue Referenzreihe)

Neue zusammenhaengende Frame-Reihe:
1. Entladen ca. 8 A
2. No-load (Last entfernt)
3. Laden ca. 20 A
4. Laden ca. 30 A

### 12.1 Bestaetigung Strommapping

`s32_le@48` verhaelt sich konsistent und vorzeichenrichtig:

- Entladen ~8 A: `s32@48 = -7821` -> `-7.821 A`
- No-load: `s32@48 = 0` -> `0 A`
- Laden ~20 A: `s32@48 = 19030` -> `19.03 A`
- Laden ~30 A: `s32@48 = 28657` -> `28.657 A`

Schlussfolgerung:
- `current_a = s32_le@48 / 1000.0` ist fuer LiTime jetzt hoch belastbar.

### 12.2 Plausibilitaet Leistung

Mit `power_w = (u32_le@8 / 1000.0) * current_a` ergeben sich plausible Werte:

- Entladen ~8 A: ca. `-103 W`
- Laden ~20 A: ca. `+254 W`
- Laden ~30 A: ca. `+385 W`

### 12.3 Beobachtete Zustandsmuster

- `u16_le@68` blieb in dieser Reihe `0` (kein Entladestopp-Bit)
- `u16_le@64` blieb stabil `10089` (factoryAh-Kandidat)
- `u16_le@62` lief dynamisch: `9802 -> 9799 -> 9804 -> 9839` (remainAh-Kandidat)
- `u32_le@100` blieb `107` (in dieser kurzen Sequenz kein Zaehlerwechsel)

### 12.4 Neuer Kandidat zu `@88`

In dieser Reihe zeigte `u16_le@88` ein reproduzierbares Muster:

- Entladen: `2`
- No-load: `0`
- Laden: `1`

Interpretation:
- `@88` ist moeglicherweise eher ein Betriebsmodusindikator (discharge/idle/charge)
  als ein reiner Balancer-Boolean.
- Balancer-Index/Bitmaske kann trotzdem in demselben oder benachbarten Bereich liegen,
  benoetigt aber gezielte Event-Korrelation pro Zelle.

Aktueller Arbeitsstand (priorisiert):
- Die Modusinterpretation ist derzeit wahrscheinlicher als eine direkte Zell-Bitmaske.
- Praktisches Muster:
  - `@88 = 2` -> Entladen
  - `@88 = 0` -> Idle/no-load
  - `@88 = 1` -> Laden

### 12.5 LiTime nahe Ladeende (ca. 4 A) - neuer Referenzpunkt

Neuer Frame bei fast vollem Akku und kleinem Ladestrom:

- `u32_le@8 = 14107` mV, `u32_le@12 = 14179` mV
- Zellwerte `@16/18/20/22 = 3508/3553/3584/3534` mV
- `s32_le@48 = 4338` -> `+4.338 A` (Skalierung bleibt konsistent)
- `u16_le@62 = 10081`, `u16_le@64 = 10089`
- `u16_le@90 = 99`, `u16_le@92 = 100`

Auffaellig am Ladeende:
- `u16_le@68`, `u16_le@72`, `u16_le@84` wechseln von `0` auf `4`
- `u16_le@88` bleibt `1` (wie andere Ladeframes)

Interpretation:
- Die Felder `@68/@72/@84` enthalten vermutlich zusaetzliche Lade-/Top-off-Statusbits,
  nicht nur das Entladestopp-Bit (`0x80`).
- Das ist ein starker Kandidat fuer "near-full/top-balance" Betriebszustand.

Praezisierung:
- Der Sprung `0 -> 4` ist mit hoher Wahrscheinlichkeit ein Ladeende-/Balancer-naher Status,
  nicht primaer ein Zellindex-Bitmaskenwert.

Zeitverhalten:
- In Folgeverlaeufen kann dieser `4`-Status wieder auf `0` zurueckfallen,
  obwohl der Akku weiterhin in hohem SOC-Bereich ist.
- Das passt zu einem transienten Event- oder Warnfenster (z. B. over-voltage/over-charge-nah),
  nicht zu einem dauerhaften Ladezustands-Bit.

### 12.6 LiTime: Ladegeraet angeschlossen mit 0 A vs. Ladegeraet getrennt

Neue direkte Gegenprobe mit zwei Frames:
- Zustand A: Ladegeraet physisch angeschlossen, aber Strom 0 A
- Zustand B: Ladegeraet getrennt

Gemeinsame Felder (identisch):
- `s32_le@48 = 0`
- `u16_le@62 = 10089`, `u16_le@64 = 10089`
- `u16_le@68 = 72`
- `u16_le@72 = 4`, `u16_le@84 = 4`
- `u16_le@88 = 0`
- `u16_le@90 = 100`, `u16_le@92 = 100`
- `u32_le@96 = 1`, `u32_le@100 = 107`

Aenderungen zwischen A und B:
- Hauptsaechlich Spannungs-/Zellwerte (`@8`, `@12`, `@16..22`)
- Keine eindeutige neue "Kabel verbunden"-Markierung in den bisher beobachteten Statusfeldern

Interpretation:
- Mit den aktuell bekannten Offsets gibt es kein robustes separates Plug-Flag,
  das "angeschlossen aber 0A" von "abgesteckt" trennen wuerde.
- Der unterscheidbare Teil liegt derzeit primaer im Spannungsverhalten,
  nicht in einem stabilen dedizierten Kabelstatus-Bit.

## 13) Bestaetigte App-Korrelation: Protect State

Hinweis aus der laufenden Validierung:
- Der in der App angezeigte Protect State war von Beginn an plausibel und konsistent.

Einordnung:
- Protect-/Schutzstatus gilt damit als inhaltlich verlässlich fuer die laufende Auswertung.
- Reverse Engineering soll weiterhin auf die Detailaufloesung einzelner Flags zielen,
  nicht auf eine Grundsatz-Infragestellung des Protect-State-Signals.
