# ESP32 BLE Battery Bridge - Detaildokumentation

Diese Datei beschreibt die aktuelle Implementierung in [bms_bt_mqtt.yaml](bms_bt_mqtt.yaml) mit Fokus auf ESPHome Web GUI und Home Assistant MQTT.

## 1) Zielbild

Das ESP32-Gateway liest BLE-BMS-Daten (PowerQueen, LiTime, Redodo) und publiziert:

- pro Batterie alle Werte als Einzel-Topics
- pro System aggregierte Pack-Werte als Einzel-Topics
- ausschliesslich unter `ble_battery/...`

## 2) ESPHome Web GUI (Home Assistant)

1. YAML in der Web-Oberflaeche oeffnen/einfuellen.
2. Secrets-Datei in der Web-Oberflaeche pflegen.
3. Validate, Compile und Install direkt aus dem Dashboard.

### 2.1 Secrets

Diese Secrets muessen vorhanden sein:

- wifi_ssid
- wifi_password
- wifi_ssid_2
- wifi_password_2
- mqtt_broker_host
- mqtt_username
- mqtt_password
- ota_password
- api_encryption_key

`mqtt_broker_host` darf sein:

- Hostname/FQDN, z. B. `mqtt.home.arpa`
- oder direkte IPv4, z. B. `192.168.1.10`

## 3) Slot- und Binding-Logik

Es gibt 6 Slots (bms1 bis bms6), jeweils mit `name`, `mac`, `enabled`.

Regeln:

- Slot aktiv, wenn `enabled = true` und `name` nicht leer.
- Bei leerer `mac` erfolgt Name-basiertes Auto-Binding und persistente Speicherung.
- Bei `enabled = false` findet kein aktives Polling/Connect fuer den Slot statt.

## 4) Scan- und Polling-Verhalten

- Scan ist global aktiv (`esp32_ble_tracker`).
- Pro Slot-Read wird Scan kurz gestoppt und danach wieder gestartet.
- `read_and_publish_all` arbeitet strikt sequenziell mit `script.wait`.

Pro Slot:

1. Effektive MAC bestimmen (statisch oder `bound_mac`)
2. `set_address(...)` am BLE-Client
3. Connect
4. FFE2-Request
5. Warten auf FFE1-Notify
6. Parsen + MQTT-Publish
7. Disconnect

## 5) MQTT-Topic-Layout

Prefix:

- `root/ble_battery`

Beispiel Batterie:

- `ble_battery/L-12100BNNA70-XXXXXX/ble_mac`
- `ble_battery/L-12100BNNA70-XXXXXX/ble_name`
- `ble_battery/L-12100BNNA70-XXXXXX/rssi`
- `ble_battery/L-12100BNNA70-XXXXXX/rssi_quality_pct`
- `ble_battery/L-12100BNNA70-XXXXXX/rssi_quality`
- `ble_battery/L-12100BNNA70-XXXXXX/voltage_mv`
- `ble_battery/L-12100BNNA70-XXXXXX/current_ma`
- `ble_battery/L-12100BNNA70-XXXXXX/power_w`
- `ble_battery/L-12100BNNA70-XXXXXX/soc`
- `ble_battery/L-12100BNNA70-XXXXXX/soh`

Beispiel System (`battery_system_name = hausspeicher`):

- `ble_battery/hausspeicher/pack_voltage_mv`
- `ble_battery/hausspeicher/pack_current_ma`
- `ble_battery/hausspeicher/pack_capacity_ah`
- `ble_battery/hausspeicher/pack_power_w`
- `ble_battery/hausspeicher/group_1/voltage_mv`
- `ble_battery/hausspeicher/group_1/current_ma`

## 6) Home Assistant

Die BMS- und Pack-Daten sind bewusst als Einzel-Topics ausgegeben. Das erleichtert MQTT-Sensoren in Home Assistant, weil jeder Messwert direkt auf einem stabilen Topic liegt.

Discovery-Sensoren werden nur fuer Slots erzeugt, die aktiviert sind (`enabled = true`) und einen Namen haben.

MQTT Discovery des ESPHome-Geraets bleibt aktiv (`discovery: true`), aber die BMS-Werte selbst werden nicht mehr als JSON-Block unter separaten Discovery/Systems-Roots geschrieben.

## 6.1) RSSI-Bedeutung

- `rssi` in dBm: je weniger negativ, desto besser.
- `rssi_quality_pct`: lineare Lesbarkeitsskala von `-100 dBm` (0%) bis `-50 dBm` (100%).
- `rssi_quality`: textuelle Stufe `excellent`, `good`, `fair`, `weak`, `very_weak`.

## 7) Group-Pack Topologie

Format von `battery_system_topology`:

- `-` trennt Seriengruppen
- Ziffern innerhalb einer Gruppe sind parallel

Beispiele:

- `123-456` -> (1||2||3) in Serie mit (4||5||6)
- `12-34-56` -> (1||2) in Serie mit (3||4) in Serie mit (5||6)

Berechnung:

- Parallelgruppe:
  - Spannung = Mittelwert der Gruppen-Slots
  - Strom = Summe
  - Kapazitaet = Summe
- Gesamtpack (Seriengruppen):
  - Spannung = Summe der Gruppenspannungen
  - Strom = Mittelwert der Gruppenstroeme
  - Kapazitaet = Minimum der Gruppenkapazitaeten

## 8) Referenzen

- Hauptkonfiguration: [bms_bt_mqtt.yaml](bms_bt_mqtt.yaml)
- Feldmapping: [bms_field_mapping.md](bms_field_mapping.md)
- Einstieg: [README.md](README.md)
