# Reverse Engineering History and Method

Stand: 2026-04-10

Dieses Dokument beschreibt den Weg, wie das aktuelle Mapping in [bms_bt_mqtt.yaml](bms_bt_mqtt.yaml) entstanden ist.
Es ist eine technische Historie, damit Entscheidungen und Annahmen spaeter nachvollziehbar bleiben.

## 1) Ziel

Aus proprietaeren BLE-Frames von LiTime- und PowerQueen-BMS sollte ein stabiler ESPHome-Parser mit MQTT/HA-Ausgabe entstehen.

Wichtige Randbedingungen:
- nur reale Feldbeobachtung, keine Blindannahmen
- gleiche Parserbasis fuer beide Marken, wenn moeglich
- robuste Fallbacks statt harter Brueche bei Varianten

## 2) Vorgehensweise

1. Rohframes aus BLE-Notify (`FFE1`) erfassen.
2. App-Werte (Spannung, Strom, SOC, SOH, Cycle, Temperaturen) zeitnah als Ground Truth danebenlegen.
3. Frames in Zustandsuebergaengen vergleichen:
	 - idle -> laden
	 - idle -> entladen
	 - discharge switch ON/OFF
4. Bytebereiche mit starker Korrelation priorisieren.
5. Parser nur fuer bestaetigte Felder festziehen, offene Felder als vorlaeufig markieren.

## 3) Schluessel-Erkenntnisse

### 3.1 Protokollgrundlage

- Feste Frame-Laenge: `105` Byte
- Checksumme: `sum(frame[0..103]) & 0xFF == frame[104]`

### 3.2 Stabile Kernfelder

Robust bestaetigt wurden unter anderem:
- Zellspannungen `@16/18/20/22`
- Temperaturen `@52/@54`
- SOC/SOH `@90/@92`
- Cycle/Discharged Counter primaer `@96/@100`

### 3.3 Status-Flags und Entlade-Sperre

- `status_flags` liegt stabil bei `u32_le@68`
- Bit `0x80` korreliert mit Entladepfad OFF
- Daraus wird das MQTT-Feld `discharge_switch` (`ON`/`OFF`) erzeugt

Wichtig: OFF bedeutet hier Entladen gesperrt, nicht zwingend Laden gesperrt.

### 3.4 Varianten (LiTime vs PowerQueen)

- Grundstruktur ist sehr nahe gleich
- Unterschiede liegen vor allem in bestimmten Status-/Reserve-Bits
- deshalb gemeinsamer Parser mit defensiver Auswertung statt zwei getrennte Parser

## 4) Parser-Strategie, die sich bewaehrt hat

- harte Frame/Checksum-Validierung vor jeder Feldauswertung
- primare Offsets fuer gesicherte Felder
- fallback fuer Zaehlerfelder bei Unplausibilitaet
- klare Trennung zwischen:
	- bestaetigt
	- vorlaeufig
	- offen

## 5) Betriebsbezogene Verbesserungen

Neben dem eigentlichen Mapping wurden fuer Stabilitaet/Usability umgesetzt:

- sequentielles Polling ueber alle Slots
- Schutz gegen ueberlappende Read-Zyklen
- MQTT Discovery robust gemacht (retained Config)
- Telemetrie non-retained
- eindeutige Entitaetsbenennung fuer Home Assistant

## 6) Was weiterhin offen ist

- exakte Semantik einzelner Felder in `62..75`
- Bedeutung von `u32@12`
- vollstaendige `equil_state`-Code-Tabelle
- finale Einordnung aller operating-mode-Codes

## 7) Reproduzierbarer Workflow fuer neue Felder

Wenn neue Bytes/Felder untersucht werden sollen:

1. Mindestens 10 Frames pro Zustand sammeln.
2. Immer App-Referenzwerte mit Zeitbezug notieren.
3. Nur ein Einflussfaktor pro Test aendern (z. B. Last, Schalter, Ladegeraet).
4. Hypothese erst nach mehreren reproduzierbaren Treffern in Parser uebernehmen.
5. Offene Punkte mit Konfidenz markieren und dokumentieren.

## 8) Ergebnis

Die aktuelle ESPHome-Implementierung ist kein theoretischer Decoder, sondern ein feldgetesteter Parser fuer den realen Betrieb mit LiTime und PowerQueen in einem Multi-Battery-Setup.
