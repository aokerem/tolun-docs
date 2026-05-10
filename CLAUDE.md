# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Tolun** is a smart aquarium automatic fish feeder system — an ESP32-S3 firmware paired with a React Native mobile app that communicate over BLE.

- **`Firmware/`** — ESP-IDF v6.0 (FreeRTOS) C firmware for ESP32-S3
- **`Mobile App/`** — React Native + Expo (TypeScript) mobile application
- **`docs/`** — System-wide specifications shared by both sides

---

## Firmware

### Build & Flash

> **Use PowerShell only. ESP-IDF v6.0 does NOT support Git Bash or MSYS.**

```powershell
# Activate ESP-IDF environment (required every new session)
C:\esp\v6.0\esp-idf\export.ps1

# Build
cd Firmware
idf.py build

# Flash to COM9
idf.py -p COM9 flash

# Serial monitor
idf.py -p COM9 monitor

# Flash + monitor in one step
idf.py -p COM9 flash monitor
```

There are no automated tests for the firmware. Testing is done manually with the serial monitor and a physical device.

### Firmware Architecture

Inter-module communication is queue-based (FreeRTOS):

```
BLE Write → cmd_queue → Command Handler → motor_queue → Motor Task (Servo)
                              ↓                              ↓
BLE Notify ← resp_queue ←────┴──────────────────────────────┘
Scheduler  → cmd_queue   (auto-feed triggers)
```

**FreeRTOS tasks** (`main/main.c` init order):
| Task | File | Priority |
|---|---|---|
| Motor Task | `drivers/motor_driver.c` | 6 |
| Command Handler | `core/command_handler.c` | 5 |
| BLE Response Task | `comm/ble_service.c` | 5 |
| Scheduler Task | `scheduler/scheduler.c` | 4 |
| Sensor Task | `drivers/sensor_driver.c` | 3 |

**Key modules:**
- `main/config/system_config.h` — single source of truth for all constants (pin numbers, limits, NVS keys, queue/task sizes)
- `core/state_manager.c` — mutex-protected state machine (`IDLE → RUNNING → IDLE`, any → `ERROR`)
- `core/command_handler.c` — validates BLE commands from `cmd_queue`, enforces state and safety rules
- `core/feed_log.c` — ring buffer (max 20 entries) persisted to NVS
- `scheduler/scheduler.c` — checks schedules every 60 s; up to 8 stored in NVS
- `comm/ble_service.c` — NimBLE GATT server; device advertises as `AquaFeeder-XXXX`

**NVS namespace:** `"tolun"` (key names in `system_config.h`)

**Safety limits enforced in firmware:**
- Portion: 1–10 per command
- Min interval between feedings: 60 s
- Daily portion limit: 30
- Motor max run time: 15 s (hardware watchdog)
- Watchdog timeout: 30 s

### GATT Services

Two custom services + two standard services:

| Service | UUID prefix | Purpose |
|---|---|---|
| Feeding Service | `6E400000-…` | Main command/response/event/status |
| Device Time Service | `6E300000-…` | `set_time` command only |
| Device Information Service | `0x180A` | FW/HW version, model |
| Elapsed Time Service | `0x183F` | Uptime |

`set_time` must go to the **Device Time Service** (`6E300001` write, `6E300002` notify), not the Feeding Service. Send once per connection.

---

## Mobile App

### Setup & Development

```powershell
cd "Mobile App"
npm install
```

BLE requires a **physical Android device** — BLE does not work in the emulator, and Expo Go is not supported. A dev client build is required.

```powershell
# Build dev client APK (only needed once, or after native dependency changes)
npx eas-cli build --platform android --profile development

# Start dev server — phone and PC on same Wi-Fi
npx expo start --dev-client

# If on different networks
npx expo start --dev-client --tunnel
```

First-time Windows setup (run once):
```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
npm install -g eas-cli
New-NetFirewallRule -DisplayName "Expo Metro" -Direction Inbound -Protocol TCP -LocalPort 8081,8082 -Action Allow
```

### Mobile App Architecture

**Provider stack** (wrapping order in `app/_layout.tsx`):
```
ThemeProvider → AppStateProvider → BleProvider → app screens
```

**BLE layer** (`src/core/ble/`):
- `BleService.ts` — singleton; wraps `react-native-ble-plx`; handles scan, connect, MTU negotiation, command send with timeout/retry, notify subscriptions
- `BleContext.tsx` — React context over `BleService`; exposes `sendCommand`, connection state, `deviceState`, `deviceInfo` (includes `schedules` and `feedLog` synced on connect)
- `BleConstants.ts` — all GATT UUIDs; supported device name prefixes (`AquaFeeder-`, `AquaLighting-`, `WallLighting-`, `HomeLighting-`)

**App state** (`src/data/AppStateContext.tsx`):
- Persists `devices`, `plans`, and `feedLogs` to `AsyncStorage` (keys prefixed `@tolun:`)
- `syncDeviceLogs` deduplicates by timestamp (±10 s window)

**On connect**, `app/_layout.tsx` runs two headless sync components:
- `DevicePlanSync` — overwrites local plans for the connected device with schedules from `deviceInfo`
- `FeedLogSync` — merges device feed log into local history

**Routing** (`app/(tabs)/`): `index` (Home), `plans`, `devices`, `settings` — device-type-specific screens are in `src/devices/Feeder/` and `src/devices/Lighting/`.

### BLE Command Protocol

All commands are JSON over the Feeding Service command characteristic. Response comes back on the response characteristic (notify). Timeout: 3 s, max 2 retries.

```json
// Send
{"cmd": "feed", "data": {"portion": 2, "portion_interval": 2000}}

// Response
{"type": "response", "cmd": "feed", "status": "success", "time": "2026-04-25 08:30:00"}

// Async event (device-initiated)
{"type": "feed_result", "mode": "manual", "portion": 2, "status": "success", "time": "..."}
```

Full command reference: [`docs/interface_spec.md`](docs/interface_spec.md)

---

## Cross-cutting Notes

- The `docs/` folder at the repo root contains specs shared by both firmware and mobile. [`docs/ble_gatt_spec.md`](docs/ble_gatt_spec.md) is the authoritative source for all GATT UUIDs — keep `BleConstants.ts` and `ble_service.c` in sync with it.
- Device name prefix in firmware (`system_config.h`: `DEVICE_NAME_PREFIX "AquaFeeder-"`) must match `DEVICE_NAME_PREFIXES` in `BleConstants.ts`.
- Wi-Fi / MQTT support is planned but not implemented. The system design doc references it, but no firmware or mobile code exists for it yet.
