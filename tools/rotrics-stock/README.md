# Rotrics Stock Artefacts

Captured from a live Nextube clock (ESP32-D0WD-V3 rev3, MAC: 34:86:5d:a9:23:f8) on 2026-04-09.

## Partition Table
| Name    | Type | Offset     | Size    |
|---------|------|------------|---------|
| nvs     | data | 0x00009000 | 20 KB   |
| otadata | data | 0x0000E000 | 8 KB    |
| app0    | app  | 0x00010000 | 4608 KB |
| app1    | app  | 0x00490000 | 4608 KB |
| spiffs  | data | 0x00910000 | 7104 KB |

## Key Findings

### ROOT CAUSE: LFS_NAME_MAX mismatch (SOLVED)
The Rotrics-supplied mklittlefs_win.exe (v0.2.3) was built with LFS_NAME_MAX=32.
The firmware's esp_littlefs component requires LFS_NAME_MAX=64.
This mismatch causes "Corrupted dir pair at {0x0, 0x1}" on mount, the firmware
then attempts to reformat the partition (wiping config.json), and falls back to
hardcoded defaults including weather_api_key="" and City="New York".

FIX: Use mklittlefs v4.0.2 from Jason2866/mklittlefs_esp32 (LFS_NAME_MAX=64).
     See tools/ folder for the correct Windows binary.

mklittlefs command:
  mklittlefs_esp32_v4.0.2_win64.exe -c [data_dir] -d 0 -b 4096 -p 256 -s 7274496 output.bin

Flash command:
  esptool --port COMx --baud 460800 write_flash 0x910000 output.bin

### OWM API Key
Rotrics original key (14e392ef37de27100ac00d560c3b0879) is dead.
Register your own free key at openweathermap.org and put it in config.json
as weather_api_key before building the LittleFS image.

Nextube Studio app.asar also hardcodes the dead key in:
  packages/renderer/dist/index.*.js
Patch this with your own key, or the app will overwrite config.json on startup.

### DST Bug
configTime(fixed_offset, 0, "pool.ntp.org") in TaskNtp.
Fix: configTzTime("CET-1CEST,M3.5.0,M10.5.0/3", "pool.ntp.org")
Requires firmware recompile (not yet patched).

### NVS
NVS contains ONLY WiFi credentials (AutoConnect) and PHY calibration data.
App config (weather key, city, timezone) is entirely LittleFS-based.

### HTTP API (all confirmed in binary)
The firmware exposes a REST API when connected to WiFi:
  POST /api/settings  - Push config JSON directly (bypasses SPIFFS entirely)
  POST /api/file/upload - Upload individual files to LittleFS
  GET  /api/ping, /api/firmwareVersion, etc.

## Files
### firmware/
- nextube_firmware_app0_full.bin - Full 4608KB app0 partition dump
- nextube_firmware_app0.bin - First 2MB (partial, earlier capture)
- partitions_raw.bin - Raw partition table (4096 bytes from 0x8000)
- strings_full.txt - All 20867 strings extracted by radare2 from full firmware

### spiffs/
- config.json - Working config (OWM key ddc308a7..., City: Kloten,CH, CEST, Celsius)
- spiffs_kloten_ch.bin - BROKEN image built with old mklittlefs v0.2.3 (LFS_NAME_MAX=32)
- spiffs_kloten_ch_lfsmax64.bin - CORRECT image built with mklittlefs v4.0.2 (LFS_NAME_MAX=64)

### tools/
- mklittlefs_esp32_v4.0.2_win64.exe - Correct mklittlefs for this firmware (LFS_NAME_MAX=64)
- libwinpthread-1.dll - Required DLL for mklittlefs_esp32_v4.0.2_win64.exe

### Root
- firmware_pseudosource.c - Annotated pseudo-source reconstructed from strings analysis
