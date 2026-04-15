# ESP32 BLE Multi-BMS to MQTT Bridge

ESPHome-based BLE gateway for LiTime/PowerQueen-style BMS devices.

The configuration reads up to 6 batteries sequentially via BLE, publishes per-battery MQTT topics, computes pack-level aggregate metrics, and exposes entities via Home Assistant MQTT Discovery.

## Highlights

- Up to 6 battery slots (`bms1` to `bms6`)
- Name-to-MAC auto-binding when static MAC is not configured
- Sequential BLE polling for higher runtime stability
- Strict frame validation (`len=105` + checksum)
- Per-battery topics and pack/system aggregate topics
- Home Assistant auto-discovery with clean entity naming

## Status

- Telemetry topics are published as non-retained (`retain=false`).
- Discovery config topics are published as retained (`retain=true`).
- Raw frame publishing is disabled by default for lower runtime load (`publish_raw_frame_hex: "false"`).
- Discharge state is published as a single field:
	- `discharge_switch` with values `ON` or `OFF`

## Quick Start

1. Open ESPHome dashboard (Home Assistant add-on or standalone).
2. Create or edit a node and paste [bms_bt_mqtt.yaml](bms_bt_mqtt.yaml).
3. Fill required secrets.
4. Set battery slots (`bmsX_name`, `bmsX_mac`, `bmsX_enabled`).
5. Set pack topology via `battery_system_topology`.
6. Validate, build, and flash.

## Required Secrets

- `wifi_ssid`
- `wifi_password`
- `wifi_ssid_2`
- `wifi_password_2`
- `wifi_ssid_3`
- `wifi_password_3`
- `mqtt_broker_host`
- `mqtt_username`
- `mqtt_password`
- `ota_password`
- `api_encryption_key`

## MQTT Topic Model

Base prefix is configurable via `mqtt_topic_prefix` (default: `ble_battery`).

Per battery:
- `ble_battery/<battery_name>/voltage_mv`
- `ble_battery/<battery_name>/current_ma`
- `ble_battery/<battery_name>/power_w`
- `ble_battery/<battery_name>/discharge_switch`
- `ble_battery/<battery_name>/operating_mode`
- `ble_battery/<battery_name>/equil_state`
- `ble_battery/<battery_name>/equil_cell` (`none`, `cell1`, ..., or CSV like `cell1,cell3` for multi-bit states)

Pack/system aggregate:
- `ble_battery/<battery_system_name>/pack_voltage_mv`
- `ble_battery/<battery_system_name>/pack_current_ma`
- `ble_battery/<battery_system_name>/pack_power_w`
- `ble_battery/<battery_system_name>/series_group_voltage_delta_mv`
- `ble_battery/<battery_system_name>/topology_reporting`
- `ble_battery/<battery_system_name>/topology_complete`

Gateway health:
- `ble_battery/<gateway_name>/read_cycle_running`
- `ble_battery/<gateway_name>/read_cycle_start_epoch`
- `ble_battery/<gateway_name>/read_cycle_last_success_epoch`
- `ble_battery/<gateway_name>/watchdog_recovery_count`

Stability behavior:
- If a read cycle is stuck for more than 180s, watchdog triggers a controlled reboot.
- If no successful cycle was published for more than 90s (while idle), watchdog triggers a recovery cycle.

## Topology Syntax

`battery_system_topology` uses slot numbers and `-` separator:

- `1-2` = 2S with one battery per series group
- `12-34` = 2S2P
- `123-456` = 2S3P
- `12-34-56` = 3S2P

Digits in one group are parallel, `-` separates series groups.

## Home Assistant Discovery

- Discovery prefix: `homeassistant` (configurable)
- Entities are created only for enabled and named battery slots
- Timestamp entities use epoch-to-localized template conversion

If old entities remain in HA, remove old retained config topics in MQTT Explorer and reload MQTT integration.

## Protocol Notes

- Frame length: `105` bytes
- Checksum: `sum(bytes[0..103]) & 0xFF == byte[104]`
- Shared parser logic works for both LiTime and PowerQueen samples

Detailed mapping is documented in [bms_field_mapping.md](bms_field_mapping.md).

## Documentation Files

- Main config: [bms_bt_mqtt.yaml](bms_bt_mqtt.yaml)
- Field mapping: [bms_field_mapping.md](bms_field_mapping.md)
- Reverse-engineering history/method: [ReverseEngineering.md](ReverseEngineering.md)
- Additional notes: [ESP32_ESPHOME_README.md](ESP32_ESPHOME_README.md)
