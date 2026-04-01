# Rotrics Nextube Official Firmware Analysis

Analysis of the official **Nextube Firmware V1.2.1** binary (released 2022-06-12),
downloaded from the Rotrics CDN:
`https://cdn.shopify.com/s/files/1/0255/0195/8216/files/Nextube_Firmware_1.2.1.zip?v=1655046714`

SHA-256: `641f0a3405d20b597dd3adc2ea875d9a5ac65d184e510fd9325fbe1606657bd8`

Analysis performed April 2026 using `esptool`, `strings`, `radare2`, and manual
binary inspection. Source code was not available — all findings are from the compiled binary.

---

## Image Header

| Field | Value |
|-------|-------|
| Magic | `0xE9` (valid ESP32) |
| ESP-IDF image version | 1 |
| Entry point | `0x40083BAC` |
| Flash size | 16 MB |
| Flash freq | 80 MHz |
| Flash mode | DIO |
| Chip | ESP32 |
| Min chip revision | v0.0 |
| Project name | `arduino-lib-builder` |
| Checksum | `0x5B` (valid) |
| Validation hash | `641f0a34...657bd8` (valid) |

The `arduino-lib-builder` project name confirms the firmware was built using
PlatformIO / Arduino framework on top of ESP-IDF, consistent with the
`previoustube` project's own build environment.

---

## Memory Segments

| Segment | File Offset | File Size | Load Address | Type |
|---------|-------------|-----------|--------------|------|
| 0 | `0x000018` | `0x664D0` (419KB) | `0x3F400020` | DROM (read-only data, Flash-mapped) |
| 1 | `0x0664F0` | `0x058AC` (22KB) | `0x3FFBDB60` | DRAM (read-write data, RAM) |
| 2 | `0x06BDA4` | `0x0426C` (17KB) | `0x40080000` | IRAM (executable, RAM) |
| 3 | `0x070018` | `0xEB274` (941KB) | `0x400D0020` | IROM (executable code, Flash-mapped) |
| 4 | `0x15B294` | `0x12FD8` (76KB) | `0x4008426C` | IRAM (executable, RAM) |
| 5 | `0x16E274` | `0x00010` (16B) | `0x50000200` | RTC_DATA |

The bulk of the application code is in **IROM** (941KB at `0x400D0020`).
Read-only data (strings, lookup tables, config keys) is in **DROM** (419KB at `0x3F400020`).

---

## FreeRTOS Tasks

The following task names were identified in the binary:

| Task Name | Notes |
|-----------|-------|
| `Task Ntp` | NTP time synchronisation |
| `Task Weather` | OpenWeatherMap API polling |
| `Task YoutubeAndBili` | YouTube / Bilibili subscriber counts |
| `Task Audio` | Sound/music visualiser |
| `Task Button` | Physical button handling |
| `Task Display` | Display rendering |
| `Task IIC` | I2C bus (RTC, SHT30 sensor) |
| `Configs` | Configuration management |
| `Led` | WS2812 LED backlight |
| `Wifi Server` | WiFi / HTTP server |

---

## NTP Implementation — The DST Bug

### Root Cause

The NTP task uses `configTime()` with a **hardcoded UTC offset of 3600 seconds (UTC+1)**
and a DST offset of 0:

```c
// Inferred from binary — this is what the firmware effectively does:
configTime(3600, 0, "pool.ntp.org");
```

The value `3600` was found at file offset `0x52BA8` / DROM vaddr `0x3F452BB0`,
immediately followed by `0x00000000` (zero DST offset), consistent with a
`configTime(tz_offset, dst_offset, ntp_server)` call structure.

The `time_zone` NVS key (stored config) holds this value, which is sent by the
Nextube Studio desktop app at upload time as:
```javascript
time_zone: new Date().getTimezoneOffset() * -60
```

However, because `configTime()` takes a **fixed integer offset**, the firmware has
no mechanism to automatically switch between winter (UTC+1) and summer (UTC+2) time.
The clock is permanently stuck at whichever offset was last written to NVS.

### NTP Server

```
pool.ntp.org
```
Found at file offset `0x1FAF7`, DROM vaddr `0x3F41FAFF`. There is no fallback server.

### Re-sync Interval

Timing constants found in the binary suggest the NTP task re-syncs approximately
every **2 hours** (7200000ms constant found at file offset `0x731984`), with an
initial sync **30 seconds to 5 minutes** after boot (consistent with official documentation).

---

## The Fix

The correct solution for automatic DST handling on ESP32 is to replace `configTime()`
with `configTzTime()` using a POSIX timezone string:

```c
// Replace this:
configTime(3600, 0, "pool.ntp.org");

// With this (handles CET/CEST automatically, forever):
configTzTime("CET-1CEST,M3.5.0,M10.5.0/3", "pool.ntp.org");
```

The POSIX string `CET-1CEST,M3.5.0,M10.5.0/3` encodes:
- `CET-1` — Central European Time, UTC+1 (winter)
- `CEST` — Central European Summer Time (UTC+2)
- `M3.5.0` — switch TO summer time: last Sunday (`5.0`) of March (`M3`) at 02:00
- `M10.5.0/3` — switch FROM summer time: last Sunday of October at 03:00

This is a one-line change. The ESP32's own `newlib` C library handles the rest,
including all future DST transitions, with no firmware updates needed.

A POSIX TZ string database for other timezones is available at:
https://github.com/nayarsystems/posix_tz_db/blob/master/zones.csv

---

## NVS Configuration Keys

The following NVS (Non-Volatile Storage) keys were found in the DROM segment.
These are the config values the Nextube Studio app writes at upload time:

| Key | Type | Notes |
|-----|------|-------|
| `time_zone` | int32 | UTC offset in seconds (the DST bug lives here) |
| `City` | string | Weather location (e.g. `Kloten, CH`) |
| `weather_api_key` | string | OpenWeatherMap API key (hardcoded in app) |
| `clock_mode` | ? | 12H / 24H / Weather mode |
| `temperature_formate` | string | `"metric"` or `"imperial"` |
| `album_switch_time` | ? | Display rotation interval |
| `album_switch_step` | ? | Display rotation step |
| `lcd_brightness` | int | Screen brightness 0–100 |
| `alarm_time` | ? | Alarm setting |
| `apps` | ? | App dock configuration |
| `theme` | ? | Display theme |

WiFi credentials are stored separately by the AutoConnect library under standard keys
(`sta.ssid`, `sta.bssid`, `ap.ssid`, etc.).

---

## Weather API

The weather feature uses **OpenWeatherMap**:

```
http://api.openweathermap.org/data/2.5/weather?q=
```

Found at file offset `0x52B2C` in DROM. The API key (`weather_api_key` NVS key)
is sent by the desktop app; the default key found in the app JS source is:
`14e392ef37de27100ac00d560c3b0879`

---

## HTTP API Endpoints

The firmware exposes a small HTTP server (AutoConnect library) for WiFi provisioning:

| Endpoint | Purpose |
|----------|---------|
| `GET /cfg/config` | Show config page |
| `POST /cfg/connect` | Submit WiFi credentials |
| `POST /cfg/del` | Delete saved credential |
| `GET /cfg/reset` | Reset device |
| `GET /cfg/result` | Connection result |
| `POST /cfg/update_act` | Firmware update action |

Default AP SSID: `nextube-ap-x`, password: `12345678`
Default AP IP: `10.10.10.1`

---

## Build Information

- **Build system**: PlatformIO / Arduino framework on ESP-IDF
- **Project name**: `arduino-lib-builder`  
- **Compile date**: `Jun 12 2022` (found in binary as `__DATE__` string)
- **Compile time**: `22:05:36`
- **Xtensa toolchain**: `xtensa-esp32-elf` (cross-compiled on Windows, path
  `C:/Users/HERRY0812/.platformio/...` visible in debug strings)
- **WiFi provisioning library**: AutoConnect
- **Display**: ST7735 (6× via SPI)
- **RTC**: NXP PCF8563 (I2C)
- **Temperature/Humidity**: Sensirion SHT30 (I2C)
- **Audio**: LTK8002D driver
- **USB-Serial**: CH340G
- **LED**: WS2812 (NeoPixel-compatible)
- **MCU**: ESP32-WROVER-E, 16MB Flash, 8MB PSRAM

Hardware revision `1.31 2022/01/19` (from PCB silkscreen, per reverse engineering notes).

---

## GPIO Mapping (from `previoustube` README, cross-checked)

| Function | GPIO |
|----------|------|
| Backlight PWM | 19 |
| SPI SCK | 12 |
| SPI MOSI | 13 |
| DC | 14 |
| Reset | 27 |
| LCD1 CS | 33 |
| LCD2 CS | 26 |
| LCD3 CS | 21 |
| LCD4 CS | 0 |
| LCD5 CS | 5 |
| LCD6 CS | 18 |

---

## Firmware Update Path

The app exposes a firmware update function via **Settings → General → Update Firmware**
in the Nextube Studio desktop app (V0.0.2). The user selects a `.bin` file and the app
uploads it over the serial connection. This means a replacement firmware (such as a
fixed build from this project) can be flashed **without any specialist tools or
command-line access** — just the existing USB cable and app.

The `/cfg/update_act` endpoint suggests OTA update over WiFi may also be possible,
though this has not been confirmed.

---

*Analysis by RSargeant1949, April 2026. Contributions and corrections welcome.*
