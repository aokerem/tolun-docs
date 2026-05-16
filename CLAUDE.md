# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

---

## Project Overview

**Tolun** is a smart home & aquarium automation platform — an ESP32-S3 firmware paired with a React Native mobile app that communicate over BLE. A single firmware codebase and a single mobile app cover **four device variants**, distinguished by their BLE advertisement name prefix:

- **AquaFeeder** — automated fish feeder with scheduling and portion control
- **AquaLighting** — aquarium light control with scheduling and color/brightness management
- **WallLighting** — wall-mounted lighting fixtures
- **HomeLighting** — whole-home / ambient lighting

The device variant is selected at firmware build time via `DEVICE_NAME_PREFIX` and `DEVICE_MODEL` in `main/config/system_config.h`. The same prefix must appear in `BleConstants.ts` (`DEVICE_NAME_PREFIXES`) on the mobile side so the app can recognise and route to the correct device screen.

- **`Firmware/`** — ESP-IDF v6.0 (FreeRTOS) C firmware for ESP32-S3
- **`Mobile App/`** — **TolunControl** — React Native + Expo (TypeScript) mobile application
- **`docs/`** — System-wide specifications shared by both sides

### Repository Structure

Firmware and mobile app are maintained in **separate git repositories**:

- `Firmware/` → its own git repo
- `Mobile App/` → its own git repo

This root directory (`tolun/`) is a monorepo-style workspace with **three independent git repositories**:

| Path | Repo | Tracks |
|---|---|---|
| `tolun/` (root) | [`tolun-docs`](https://github.com/aokerem/tolun-docs) | `docs/`, `3d/`, `CLAUDE.md`, `VERSIONING.md` — paylaşılan spec'ler ve dokümantasyon |
| `tolun/Firmware/` | [`aquarium-firmware`](https://github.com/aokerem/aquarium-firmware) | ESP32-S3 firmware (ESP-IDF v6.0) |
| `tolun/Mobile App/` | [`aquarium-mobile`](https://github.com/aokerem/aquarium-mobile) | TolunControl React Native app |

`Firmware/` ve `Mobile App/` root repo'nun `.gitignore`'unda olduğundan tolun-docs onları "alt klasör" olarak görmez. Her commit, branch ve tag ilgili repo'da bağımsız yönetilir.

> **IMPORTANT:** Git komutu çalıştırmadan önce hangi repo'da olduğundan emin ol. Doküman/spec değişiklikleri tolun root'ta, firmware değişiklikleri `Firmware/`'de, mobil değişiklikleri `Mobile App/`'te commit edilir. Protokol değişikliği gibi her iki tarafı etkileyen iş için **üç ayrı commit** gerekir (firmware + mobile + spec).

> **🔴 UNUTMA — tolun-docs senkronizasyonu:** Her firmware veya mobile commit'inden sonra ilgili changelog (`docs/firmware_changelog.md` / `docs/mobileapp_changelog.md`) ve gerekirse spec dosyaları (`docs/ble_gatt_spec.md`, `docs/interface_spec.md` vb.) tolun root'ta da commit + push edilmelidir. `Firmware/` ve `Mobile App/`'teki `post-commit` hook'u commit sonrası bu değişiklikler bekliyorsa terminale uyarı basar — sarı uyarı gördüğünde:
>
> ```bash
> cd ..   # tolun root
> git add -A && git commit -m "docs: <açıklama>" && git push
> ```
>
> Bu sayede sub-repo commit/release'leri ile paylaşılan doküman değişiklikleri senkron kalır.

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

**FreeRTOS tasks** (priorities defined in `main/config/system_config.h`):
| Task | File | Priority |
|---|---|---|
| Motor Task | `drivers/motor_driver.c` | 6 |
| Command Handler | `core/command_handler.c` | 4 |
| BLE Response Task | `comm/ble_service.c` | 4 |
| Scheduler Task | `scheduler/scheduler.c` | 4 |
| Sensor Task | `drivers/sensor_driver.c` | 3 |
| OLED Display Task | `drivers/oled_display.c` | 2 |

**Key modules:**
- `main/config/system_config.h` — single source of truth for all constants (pin numbers, limits, NVS keys, queue/task sizes, `DEVICE_NAME_PREFIX`, `DEVICE_MODEL`)
- `core/state_manager.c` — mutex-protected state machine (`IDLE → RUNNING → IDLE`, any → `ERROR`)
- `core/command_handler.c` — validates BLE commands from `cmd_queue`, enforces state and safety rules
- `core/feed_log.c` — ring buffer (max 20 entries) persisted to NVS; resets daily at 00:00 UTC
- `core/lighting_log.c` — ring buffer (max 20 entries) persisted to NVS for lighting sessions
- `core/device_stats.c` — first BT connection timestamp + cumulative uptime, persisted to NVS every 5 min
- `scheduler/scheduler.c` — checks schedules every 60 s; up to 5 stored in NVS
- `comm/ble_service.c` — NimBLE GATT server; device advertises as `<DEVICE_NAME_PREFIX><XXXX>` where `XXXX` is the last 4 hex digits of the MAC (e.g. `AquaLighting-AB12`)

**NVS namespace:** `"tolun"` (key names in `system_config.h`)

**Safety limits enforced in firmware:**
- Portion: 1–10 per command
- Min interval between feedings: 60 s
- Daily portion limit: 30
- Motor max run time: 20 s (hardware watchdog)
- Watchdog timeout: 30 s

### GATT Services

One custom service + three Bluetooth SIG standard services:

| Service | Service UUID | Characteristic | Char UUID | Properties |
|---|---|---|---|---|
| **Feeding Service** | `6E400000-B5A3-F393-E0A9-E50E24DCCA9E` | Command | `6E400001-…` | Write |
| | | Response | `6E400002-…` | Notify |
| | | Event | `6E400003-…` | Notify |
| | | Status | `6E400004-…` | Read + Notify |
| **Device Time Service** | `0x1847` (standard) | Time Command | `6E300001-…` | Write |
| | | Time Response | `6E300002-…` | Notify |
| **Device Information Service** | `0x180A` (standard) | Device Name | `0x2A00` | Read |
| | | Manufacturer | `0x2A29` | Read |
| | | Model Number | `0x2A24` | Read |
| | | FW Revision | `0x2A26` | Read |
| | | HW Revision | `0x2A27` | Read |
| **Elapsed Time Service** | `0x183F` (standard) | Elapsed Time | `0x2BF2` | Read + Notify |

All `6E4xxxxx` and `6E3xxxxx` UUIDs share the suffix `-B5A3-F393-E0A9-E50E24DCCA9E`.

**MTU strategy** — peripheral advertises preferred MTU 512 (`CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=512`). Android negotiates up to 517, but `BluetoothGatt.MAX_ATTR_LEN` caps every single ATT transaction at **512 bytes**. To stay under that cap, all bulk data (schedules, feed log, lighting log) is fetched **per-index** via JSON commands on `6E400001` — there are no bulk-read characteristics. An earlier `Feed Log` (`6E400005`) and `Lighting Log` (`6E400006`) read characteristic pair existed but hit the 512 B cap and was removed.

**DIS contract** — `Model Number` (`0x2A24`) must match `DEVICE_MODEL` in `system_config.h` and the BLE advertisement prefix. `FW Revision` reflects `FIRMWARE_VERSION`, `HW Revision` reflects `HARDWARE_VERSION`. The mobile app reads `fwVersion` / `hwVersion` from DIS, **not** from the status JSON payload.

`set_time` must go to the **Device Time Service** (service `0x1847`; write char `6E300001-…`, notify char `6E300002-…`), not the Feeding Service. Send once per connection.

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
- `BleService.ts` — singleton; wraps `react-native-ble-plx`; handles scan, connect, MTU negotiation (target 512), command send with 3 s timeout + 2 retries, notify subscriptions, and per-index pagination of schedules + logs
- `BleContext.tsx` — React context over `BleService`; exposes `sendCommand`, connection state, `deviceState`, `deviceInfo` (includes `fwVersion` / `hwVersion` read from DIS, plus `schedule_count` / `feed_log_count` / `lighting_log_count` from status — each list is then fetched per-index)
- `BleConstants.ts` — all GATT UUIDs; supported device name prefixes (`AquaFeeder-`, `AquaLighting-`, `WallLighting-`, `HomeLighting-`); `deviceTypeFromName()` maps each prefix to a user-facing device type label

**App state** (`src/data/AppStateContext.tsx`):
- Persists `devices`, `plans`, and `feedLogs` to `AsyncStorage` (keys prefixed `@tolun:`)
- `syncDeviceLogs` deduplicates by timestamp (±10 s window)

**On connect**, `BleService.ts` performs sequential reads (parallel `Promise.all` is deliberately avoided to stay under NimBLE's `GATT_MAX_PROCS=4` ceiling):
1. `readDIS()` → 5 DIS reads (`deviceName`, `manufacturer`, `modelNumber`, `fwVersion`, `hwVersion`)
2. `readUptime()` → Elapsed Time Service (`0x2BF2`)
3. `syncTime()` → writes current epoch + tz offset on `6E300001`
4. `readStatus()` → reads status char (`6E400004`). Status JSON carries `schedule_count` / `feed_log_count` / `lighting_log_count`. `readStatus` then chains:
   - `fetchSchedules(count)` → loops `get_schedule {index:i}` for i in 0…count-1
   - `fetchFeedLog(count)` → loops `get_feed_log_entry {index:i}`
   - `fetchLightingLog(count)` → loops `get_lighting_log_entry {index:i}`

Each per-index response is ~150–250 B, well under the 512 B ATT cap. The three `fetch*` methods use `*InFlight` reentrancy guards so repeated status notifies (e.g. after `lighting_off`) don't spawn overlapping fetch loops. `lastScheduleCount` / `lastFeedLogCount` / `lastLightingLogCount` cache last-seen counts — if a notify arrives with the same count, the corresponding fetch is skipped. All three cached values are reset to `-1` on disconnect to guarantee a full fetch on reconnect.

Headless sync components in `app/_layout.tsx`: `DevicePlanSync` (overwrites local plans with fetched schedules), `FeedLogSync` (merges feed logs with ±10 s timestamp deduplication), `LightingLogSync` (same dedup model for lighting sessions; manual on/off also writes a local entry immediately without waiting for BLE).

**Routing** (`app/(tabs)/`): `index` (Home), `plans`, `devices`, `settings` — device-type-specific screens are in `src/devices/Feeder/` and `src/devices/Lighting/`.

### BLE Command Protocol

All commands are JSON over the Feeding Service command characteristic (`6E400001-…`). Responses come back on the response characteristic (`6E400002-…`). Timeout: 3 s, max 2 retries.

Full command reference (all commands, payloads, validation rules, error codes): [`docs/interface_spec.md`](docs/interface_spec.md)

---

## Release Akışı

Firmware ve Mobile App ayrı repolarda bağımsız olarak versiyonlanır. Her release için şu adımlar sırayla yapılır:

**Commit aşaması** (günlük iş — commitler birikir):
```
feat: yeni özellik
fix: hata düzeltme
feat: vX.Y.Z — açıklama   (release commit'i)
```
Format Conventional Commits. Mesajda `vX.Y.Z` varsa `prepare-commit-msg` hook'u ilgili dosyadaki versiyonu otomatik bump eder ve commit'e ekler — Firmware: `main/config/system_config.h` → `FIRMWARE_VERSION`; Mobile App: `package.json` → `version`. Versiyon içermeyen commit'lerde (ara çalışma) hook no-op kalır. Hook'lar `scripts/git-hooks/prepare-commit-msg`'de tanımlı, `core.hooksPath` ile aktif.

> **Yeni clone sonrası bir kez:** `git config core.hooksPath scripts/git-hooks` (her iki repo'da). Yapılmazsa otomatik versiyon bump çalışmaz.

**Release aşaması** (`commit` komutu verildiğinde 1–2, `push` komutu verildiğinde 3–5):

1. `docs/firmware_changelog.md` veya `docs/mobileapp_changelog.md` güncelle
2. `git commit -m "feat: vX.Y.Z — kısa açıklama"` — hook versiyonu otomatik bump eder
3. `git tag vX.Y.Z`
4. `git push origin main && git push origin vX.Y.Z`
5. GitHub Release oluştur (tag üzerinden, changelog bölümünü release notuna ekle)

Versiyonlama kurallarının tamamı: [`VERSIONING.md`](VERSIONING.md)

---

## Cross-cutting Notes

- The `docs/` folder at the repo root contains specs shared by both firmware and mobile. [`docs/ble_gatt_spec.md`](docs/ble_gatt_spec.md) is the authoritative source for all GATT UUIDs — keep `BleConstants.ts` and `ble_service.c` in sync with it.
- **Changelogs live in `docs/`** — [`docs/firmware_changelog.md`](docs/firmware_changelog.md) and [`docs/mobileapp_changelog.md`](docs/mobileapp_changelog.md) are the canonical changelog files.
- **Device variant invariants** — for each build, all four of these must agree:
  1. `DEVICE_NAME_PREFIX` in `system_config.h` (e.g. `"AquaLighting-"`)
  2. `DEVICE_MODEL` in `system_config.h` (e.g. `"AquaLighting"`) — exposed via DIS `0x2A24`
  3. The matching entry in `DEVICE_NAME_PREFIXES` in `BleConstants.ts`
  4. The corresponding label returned by `deviceTypeFromName()` in `BleConstants.ts`
  Mismatch causes the device to either not appear in scan results or be misclassified in the UI.
- **No bulk-read log/schedule characteristics** — every schedule, feed log entry, and lighting log entry travels through the `Command` (`6E400001`) / `Response` (`6E400002`) pair via per-index JSON commands. This is intentional: Android's `BluetoothGatt.MAX_ATTR_LEN` 512 B cap rules out single-shot bulk reads for these payloads.
- Wi-Fi / MQTT support is planned but not implemented. The system design doc references it, but no firmware or mobile code exists for it yet.
