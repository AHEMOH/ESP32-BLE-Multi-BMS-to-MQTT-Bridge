# ESP32 BLE Multi-BMS to MQTT Bridge

ESPHome-based BLE gateway for LiTime, PowerQueen, and Redodo BMS devices. Reads up to 6 batteries sequentially via BLE, publishes per-battery MQTT topics, computes pack-level aggregate metrics, and exposes entities via Home Assistant MQTT Discovery.

> Based on `dmytro-tsepilov/pq_bms_bluetooth`. Special thanks to the original project; this repository is a full ESPHome rewrite with additional reverse-engineering and protocol enhancements.

## Quick Start

1. Open ESPHome dashboard (Home Assistant add-on or standalone).
2. Create a new node, paste `esp32-ble-battery-bridge.yaml`.
3. Fill in `secrets.yaml` with WiFi, MQTT, and battery slot values (see below).
4. Validate, compile, and flash.

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
| `bms1_name` … `bms6_name` | Battery name (MQTT topic key); leave empty to disable slot |
| `bms1_mac` … `bms6_mac` | BLE MAC address; leave empty to auto-bind by name |

## Slot Configuration

Six slots (`bms1`–`bms6`) are available. By default, only `bms1` and `bms2` are wired to `secrets.yaml`. Slots 3–6 are disabled with empty string defaults directly in the substitutions block.

### Active slots (bms1 / bms2)

Values come from `secrets.yaml` (gitignored — never committed):

```yaml
# secrets.yaml
bms1_name: "P-12100BNNA70-XXXXXX"   # MQTT topic key; empty = slot inactive
bms1_mac:  "XX:XX:XX:XX:XX:XX"      # BLE address; empty = auto-bind by name

bms2_name: "L-12100BNNA70-XXXXXX"
bms2_mac:  "XX:XX:XX:XX:XX:XX"
```

### Inactive slots (bms3–bms6)

These have hardcoded empty strings in `esp32-ble-battery-bridge.yaml` and are always disabled unless you explicitly activate them:

```yaml
# esp32-ble-battery-bridge.yaml — substitutions block
bms3_name: ""      ← disabled by default
bms3_mac:  ""
bms3_enabled: "false"
```

### Activating an additional slot

ESPHome has no optional-secret fallback — a missing `!secret` key is a compile error. To activate e.g. slot 3:

1. Add values to `secrets.yaml`:
   ```yaml
   bms3_name: "P-12100BNNA70-XXXXXX"
   bms3_mac:  "XX:XX:XX:XX:XX:XX"
   ```
2. Change two lines in the `substitutions` block of the main YAML:
   ```yaml
   bms3_name: !secret bms3_name   # was: ""
   bms3_mac:  !secret bms3_mac    # was: ""
   bms3_enabled: "true"           # was: "false"
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

**Unit convention:** whole-battery voltages and pack voltages in **V**; battery and pack currents in **A**; cell voltages and any delta/spread in **mV**.

**Per battery** (`ble_battery/<battery_name>/`):

| Topic | Description |
|-------|-------------|
| `voltage_v`, `current_a`, `power_w` | Electrical state (terminal/pole voltage) |
| `pack_voltage_v` | BMS-internal cell-sum voltage (drops to ~0 when discharge MOSFET is off) |
| `soc`, `soh` | State of charge / health (%) |
| `cell1_mv`–`cell4_mv` | Per-cell voltages |
| `cell_min_mv`, `cell_max_mv` | Weakest / strongest cell voltage |
| `cell_delta_mv`, `cell_delta_status` | Cell spread and balance status |
| `cell_voltage_status` | `critical_low` / `low` / `normal` / `high` (based on weakest cell) |
| `cell_temp`, `mosfet_temp` | Temperatures (°C) |
| `remain_ah`, `factory_ah` | Remaining / nominal capacity |
| `discharge_switch` | `ON` / `OFF` (derived from status_flags bit 0x80) |
| `equil_state`, `equil_cell` | Balancer state; cell names as CSV e.g. `cell1,cell3` |
| `protect_state` | Protection-state code (raw integer) |
| `operating_mode_raw`, `operating_mode` | Mode raw code and text label (`idle` / `charge` / `discharge` / `voltage_protection` / `unknown`) |
| `cycle_count`, `discharge_ah_count` | Lifetime counters |
| `rssi`, `rssi_quality_pct`, `rssi_quality` | BLE signal |
| `ble_mac`, `ble_name` | BLE device info |

**Pack / system aggregate** (`ble_battery/<battery_system_name>/`):

`pack_voltage_v`, `pack_voltage_internal_v`, `pack_current_a`, `pack_power_w`, `pack_soc`, `pack_capacity_ah`, `pack_remain_ah`, `series_group_voltage_delta_mv`, `slot_voltage_delta_mv`, `pack_cell_delta_max_mv`, `pack_cell_min_mv`, `pack_cell_balancer_status`, `pack_series_balancer_status`, `operating_mode_raw`, `operating_mode` (max raw across active batteries → text), `victron_balancer_active`, `victron_balancer_alarm`, `victron_balancer_status`, `group_1/voltage_v`, `group_1/current_a` … (per-group sub-topics), `topology`, `topology_reporting`, `topology_complete`

`pack_voltage_v` is the pole/terminal sum (what loads see); `pack_voltage_internal_v` is the BMS-internal cell-sum aggregation. Compare them to spot MOSFET-off events or large drops under load.

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
| `operating_mode` | 88 | u16 LE | code | `0`=idle, `1`=charge, `2`=discharge, `4`=voltage_protection (see legend below) |
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

### Operating Mode Codes

| Code | Label                | Notes |
|------|----------------------|-------|
| 0    | `idle`               | |
| 1    | `charge`             | |
| 2    | `discharge`          | |
| 4    | `voltage_protection` | Observed when a 12 V LFP battery in a 24 V series string hit ~15 V — generic voltage-rail protection trip (over- or under-voltage). |
| else | `unknown`            | Not yet identified — please report. |

The pack-aggregate device exposes `operating_mode_raw` / `operating_mode` derived from `max()` of the per-battery raw codes, so a single battery in protection surfaces at the pack level.

### Victron Battery Balancer (24 V) Integration

If you have a Victron BatteryBalancer wired across two 12 V batteries in series, the pack device exposes three entities that mirror the real device's behavior:

| Entity                       | Meaning |
|------------------------------|---------|
| `victron_balancer_active`    | `true` while pack > 27.3 V (balancer is shunting) |
| `victron_balancer_alarm`     | `true` once group spread > 200 mV; clears at < 140 mV or pack < 26.6 V (matches Victron alarm relay hysteresis) |
| `victron_balancer_status`    | `n/a` / `below_activation` / `balancing_idle` / `balancing_active` / `alarm` |

Spread input: `series_group_voltage_delta_mv`. Only meaningful when `battery_system_topology` is exactly two series groups (e.g. `1-2`, `12-34`, `123-456`); reports `n/a` otherwise. Sources: [Victron Battery Balancer product page](https://www.victronenergy.com/batteries/battery-balancer).

### Parser Fallback Rules

- `cycle_count`: primary `u32@96`; falls back to `u16@66` if primary is 0 or ≥ 1,000,000
- `discharge_ah_count`: primary `u32@100`; falls back to `u32@68` if primary is 0 or ≥ 100,000,000

### Open / Unconfirmed Fields

The following are partially reverse-engineered and may vary by firmware version:

- `u32@12` — secondary voltage; exact semantics unclear
- `equil_state@84` bitmask — heuristic cell-index mapping, not final
- `operating_mode@88` full code table — `0/1/2/4` mapped (see Operating Mode Codes); other values not yet identified
- Flag details in bytes 62–75 and 80–83

Protocol confirmed on LiTime and PowerQueen samples; Redodo assumed compatible (same family).
