# Rotrics Stock Artefacts

Captured from a live Nextube clock (ESP32-D0WD-V3 rev3, MAC: 34:86:5d:a9:23:f8) on 2026-04-09.

## Partition Table
| Name    | Type | Offset     | Size       |
|---------|------|------------|------------|
| nvs     | data | 0x00009000 | 20 KB      |
| otadata | data | 0x0000E000 | 8 KB       |
| app0    | app  | 0x00010000 | 4608 KB    |
| app1    | app  | 0x00490000 | 4608 KB    |
| spiffs  | data | 0x00910000 | 7104 KB    |

## Files
- `firmware/nextube_firmware_app0.bin` — 2MB read of app0 partition (0x10000). Full firmware not captured due to flash corruption beyond 2MB.
- `firmware/partitions_raw.bin` — Raw partition table (0x8000, 4096 bytes).
- `spiffs/config.json` — Working config with active OWM key (ddc308a7...), City: Kloten,CH, Celsius, time_zone: 7200 (CEST).
- `spiffs/spiffs_kloten_ch.bin` — LittleFS image (7274496 bytes) ready to flash to 0x910000. Built with mklittlefs -b 4096 -p 256 -s 7274496.

## Key Findings
- OWM API key is NOT hardcoded in firmware — it is read from /config.json on SPIFFS/LittleFS at boot.
- After first boot the firmware caches config into NVS. Erasing NVS (0x9000, 0x5000) forces re-read from SPIFFS.
- Rotrics' original OWM key (14e392ef37de27100ac00d560c3b0879) is dead. Register your own at openweathermap.org.
- The Nextube Studio app hardcodes the dead key in its asar renderer bundle. Patch packages/renderer/dist/index.*.js in app.asar to use your own key.
- SPIFFS flash address: 0x910000. Flash command: esptool --port COMx --baud 460800 write_flash 0x910000 spiffs_kloten_ch.bin
- time_zone field: seconds offset from UTC. CEST = 7200, CET = 3600. App computes this dynamically from PC clock.
