# ESP32 BLE Multi-BMS to MQTT Bridge

ESPHome-based BLE gateway for LiTime, PowerQueen, and Redodo BMS devices. Reads up to 6 batteries sequentially via BLE, publishes per-battery MQTT topics, computes pack-level aggregate metrics, and exposes entities via Home Assistant MQTT Discovery.

## Quick Start

1. Open ESPHome dashboard (Home Assistant add-on or standalone).
2. Create a new node, paste `esp32-ble-battery-bridge.yaml`.
3. Fill in `secrets.yaml` with required credentials.
4. Set battery slots in the `substitutions` section.
5. Validate, compile, and flash.

## Required Secrets

| Key | Description |
|-----|-------------|
| `wifi_ssid` / `wifi_password` | Primary WiFi |
| `wifi_ssid_2` / `wifi_password_2` | Fallback WiFi |
| `wifi_ssid_3` / `wifi_password_3` | Second fallback WiFi |
| `mqtt_broker_host` | Hostname, FQDN, or IP of MQTT broker |
| `mqtt_username` / `mqtt_password` | MQTT credentials |
| `ota_password` | OTA update password |
| `api_encryption_key` | ESPHome API key |

## Slot Configuration

Six slots (`bms1`–`bms6`) are configured in the `substitutions` block:

```yaml
bms1_name:    "P-12100BNNA70-XXXXXX"   # MQTT topic key; empty = slot inactive
bms1_mac:     "XX:XX:XX:XX:XX:XX"      # BLE address; empty = auto-bind by name
bms1_enabled: "true"
```

A slot is active when `enabled: "true"` **and** `name` is not empty.

**Auto-binding:** If `mac` is left empty, the BLE tracker resolves the device by name on first scan and stores the MAC in NVS flash. It survives reboots. Names must start with `P-`, `L-`, or `R-` (PowerQueen, LiTime, Redodo families).

## Pack Topology

`battery_system_topology` encodes series/parallel arrangement using slot numbers:

| Value | Meaning |
|-------|---------|
| `"1-2"` | 2S, one battery per group |
| `"12-34"` | 2S2P |
| `"123-456"` | 2S3P |
| `"12-34-56"` | 3S2P |

Digits within one group are parallel; `-` separates series groups.

**Aggregation rules:**
- Parallel group voltage = average of slot voltages
- Parallel group current = sum of slot currents
- Series pack voltage = sum of group voltages
- Series pack current = average of group currents
- Pack capacity = minimum of group capacities

## MQTT Topic Model

Base prefix: `ble_battery` (configurable via `mqtt_topic_prefix`).

**Per battery** (`ble_battery/<battery_name>/`):

| Topic | Description |
|-------|-------------|
| `voltage_mv`, `current_ma`, `power_w` | Electrical state |
| `pack_voltage_mv` | Secondary voltage reading |
| `soc`, `soh` | State of charge / health (%) |
| `cell1_mv`–`cell4_mv` | Per-cell voltages |
| `cell_min_mv`, `cell_max_mv` | Weakest / strongest cell voltage |
| `cell_delta_mv`, `cell_delta_status` | Cell spread and balance status |
| `cell_voltage_status` | `critical_low` / `low` / `normal` / `high` (based on weakest cell) |
| `cell_temp`, `mosfet_temp` | Temperatures (°C) |
| `remain_ah`, `factory_ah` | Remaining / nominal capacity |
| `discharge_switch` | `ON` / `OFF` (derived from status_flags bit 0x80) |
| `equil_state`, `equil_cell` | Balancer state; cell names as CSV e.g. `cell1,cell3` |
| `protect_state`, `operating_mode` | Protection / mode status |
| `cycle_count`, `discharge_ah_count` | Lifetime counters |
| `rssi`, `rssi_quality_pct`, `rssi_quality` | BLE signal |
| `ble_mac`, `ble_name` | BLE device info |

**Pack / system aggregate** (`ble_battery/<battery_system_name>/`):

`pack_voltage_mv`, `pack_current_ma`, `pack_power_w`, `pack_soc`, `pack_capacity_ah`, `pack_remain_ah`, `series_group_voltage_delta_mv`, `slot_voltage_delta_mv`, `pack_cell_delta_max_mv`, `pack_cell_min_mv`, `pack_cell_balancer_status`, `pack_series_balancer_status`, `group_1/voltage_mv` … (per-group sub-topics), `topology`, `topology_reporting`, `topology_complete`

**Gateway health** (`ble_battery/esp32-battery-bridge/`):

`uptime_s`, `ip_address`, `wifi_ssid`, `wifi_connected`, `mqtt_connected`, `read_cycle_running`, `read_cycle_last_success_epoch`, `watchdog_recovery_count`, `last_error`, `last_warning`

Telemetry topics: non-retained. Discovery config topics: retained.

## Home Assistant Discovery

MQTT Discovery is published automatically on MQTT connect and refreshed every 5 minutes. Entities are created only for active (enabled + named) slots.

To remove stale entities: delete the retained discovery config topics in an MQTT explorer and reload the MQTT integration in HA.

## BLE Frame Protocol

### Frame Structure

- Length: **105 bytes**, little-endian
- Checksum: `sum(bytes[0..103]) & 0xFF == byte[104]`
- Command sent to request data: `0x00 0x00 0x04 0x01 0x13 0x55 0xAA 0x17` (written to characteristic `FFE2`)
- Data received on characteristic `FFE1` (notify)

### Field Offsets

| Field | Offset | Type | Unit | Notes |
|-------|-------:|------|------|-------|
| `pack_voltage_mv` | 8 | u32 LE | mV | State-dependent; may read low when discharge is off |
| `voltage_mv` (alt) | 12 | u32 LE | mV | Variant-specific secondary voltage |
| `cell1_mv` | 16 | u16 LE | mV | |
| `cell2_mv` | 18 | u16 LE | mV | |
| `cell3_mv` | 20 | u16 LE | mV | |
| `cell4_mv` | 22 | u16 LE | mV | |
| `current_ma` | 48 | s32 LE | mA | Negative = discharge, positive = charge |
| `cell_temp` | 52 | u16 LE | °C | |
| `mosfet_temp` | 54 | u16 LE | °C | |
| `remain_ah_raw` | 62 | u16 LE | 0.01 Ah | Divide by 100 for Ah |
| `factory_ah_raw` | 64 | u16 LE | 0.01 Ah | Divide by 100 for Ah |
| `status_flags` | 68 | u32 LE | bitfield | Bit `0x80` = discharge path locked (OFF) |
| `protect_state` | 76 | u8 (LSB of u32) | code | |
| `equil_state` | 84 | u8 (LSB of u32) | bitmask | `1`=cell1, `2`=cell2, `4`=cell3, `8`=cell4 (heuristic) |
| `operating_mode` | 88 | u16 LE | code | `0`=idle, `1`=charge, `2`=discharge |
| `soc` | 90 | u8 (LSB of u16) | % | |
| `soh` | 92 | u8 (LSB of u16) | % | |
| `cycle_count` | 96 | u32 LE | count | Primary; fallback to u16@66 if implausible |
| `discharge_ah_total` | 100 | u32 LE | raw | Lifetime Ah counter |
| **checksum** | **104** | **u8** | | `sum(bytes[0..103]) & 0xFF` |

### Balancer Thresholds

**Cell delta** (within one battery, tuned for passive single-cell balancers on LiFePO4):

| Range | Status |
|-------|--------|
| ≤ 10 mV | `excellent` |
| ≤ 25 mV | `good` |
| ≤ 40 mV | `normal` |
| ≤ 60 mV | `warning` |
| > 60 mV | `critical` |

**Cell voltage status** (based on weakest cell, LiFePO4 4S):

| Weakest cell | Status |
|-------------|--------|
| ≥ 3500 mV | `high` (charging / ≥ 90% SoC) |
| ≥ 3050 mV | `normal` (typical working range) |
| ≥ 2900 mV | `low` (approx. ≤ 15% SoC) |
| < 2900 mV | `critical_low` (near BMS cutoff) |

**Series group delta** (between 12V packs):

| Range | Status |
|-------|--------|
| ≤ 20 mV | `good` |
| ≤ 50 mV | `light` |
| ≤ 120 mV | `warning` |
| > 120 mV | `critical` |

### Parser Fallback Rules

- `cycle_count`: primary `u32@96`; falls back to `u16@66` if primary is 0 or ≥ 1,000,000
- `discharge_ah_count`: primary `u32@100`; falls back to `u32@68` if primary is 0 or ≥ 100,000,000

### Open / Unconfirmed Fields

The following are partially reverse-engineered and may vary by firmware version:

- `u32@12` — secondary voltage; exact semantics unclear
- `equil_state@84` bitmask — heuristic cell-index mapping, not final
- `operating_mode@88` full code table — `0/1/2` confirmed; higher values unknown
- Flag details in bytes 62–75 and 80–83

Protocol confirmed on LiTime and PowerQueen samples; Redodo assumed compatible (same family).
