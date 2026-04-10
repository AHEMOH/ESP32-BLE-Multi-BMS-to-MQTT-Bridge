# Draft Message to Original Developer

Hi <developer_name>,

I wanted to share a quick update and say thank you for your original work on this BLE BMS project.

I have been running additional reverse engineering and field tests with both PowerQueen and LiTime batteries, and I found several useful mappings and behavior details that seem stable across both variants.

Key updates from my testing:
- Confirmed 105-byte frame structure with checksum validation.
- Confirmed stable offsets for SOC/SOH, cell temperatures, per-cell voltages, and cycle/discharge counters.
- Confirmed status flag behavior for discharge path lock (bit-based interpretation).
- Added safer parsing with fallback logic for variant-dependent counter fields.
- Improved runtime robustness for BLE polling and concurrent app usage.

I also rewrote my implementation into an ESPHome-first architecture (BLE -> MQTT -> Home Assistant), including:
- Multi-battery support (up to 6 slots)
- Per-battery topics + pack-level aggregate topics
- Home Assistant MQTT Discovery
- Cleaner entity naming and topic model
- Better operational stability in long-running use

Your previous work and protocol direction helped a lot as a baseline, so thank you again.
If you are interested, I can share a concise mapping table and example frames with confidence notes for each field.

Best regards,
<your_name>
