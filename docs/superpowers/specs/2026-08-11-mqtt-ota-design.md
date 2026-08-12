# MQTT, Home Assistant Integration, and OTA Updates

Date: 2026-08-11

## Context

The Christmas Lights controller (`src/main.cpp`) currently:
- Connects to WiFi and serves a raw `WiFiServer`-based web UI for picking a color/pattern (`RadioProcessor`, `UpdatePalette`).
- Has no MQTT, no Home Assistant integration, and no OTA update capability — every firmware change requires a physical USB connection.
- Has a periodic WiFi check in `loop()`, but the reconnect path (`Connect_to_Wifi()`) blocks indefinitely on failure, freezing LED animation and the web server during an outage.

A sibling project, `AirSensor` (`/Users/edinel/src/Air Quality/AirSensor/src/main.cpp`), already has MQTT + Home Assistant discovery + an MQTT-triggered `ArduinoOTA` flow. This design ports that pattern to the lights controller, adapted for its continuous (non-sleeping) operating model.

## Goals

1. Add MQTT connectivity, reusing the existing Home Assistant broker.
2. Publish full Home Assistant discovery for the light pattern, bidirectionally controllable from HA.
3. Add an MQTT-triggered OTA update mode, safe for a device that runs continuously and can't rely on a `"cancel"` message always arriving.
4. Harden the existing WiFi reconnect logic so a dropped connection doesn't hang `loop()`.

## Design

### 1. MQTT / Home Assistant integration

- Add `knolleary/PubSubClient` to `platformio.ini` `lib_deps` (matches `AirSensor`'s `platformio.ini`).
- Extend `include/arduino-secrets.h` with `mqttServer`, `mqttUser`, `mqttPass` (currently only has `ssid`/`pass`). Reuses the same broker AirSensor already talks to (`homeassistant.local.solace.org`).
- Device identity: change `hostname` from `"BackYard-Xmas-Arduino"` to `"twinkle-back"`, and derive `device_id = "twinkle_back"` for discovery `unique_id`s, following AirSensor's `DEVICE_ID` convention.
- Publish an MQTT discovery **`select`** entity:
  - Discovery topic: `homeassistant/select/<device_id>/pattern/config`
  - Options: `off, red, blue, green, yellow, cyan, orange, white, gold, hanukkah, nordic` — the same values `RadioProcessor`/`UpdatePalette` already use for `gButtonClicked`.
  - State topic: `<hostname>/select/pattern/state` — published whenever `gButtonClicked` changes, regardless of source (web UI or HA).
  - Command topic: `<hostname>/select/pattern/set` — on receipt, sets `gButtonClicked` to the payload and republishes state. This is the same assignment the web handler already performs, so `UpdatePalette()` needs no changes.
- A single `<hostname>/cmd` topic (mirroring AirSensor's `CMD_TOPIC`) carries OTA control commands (see below), separate from the pattern select topic.
- `reconnectMQTT()`: ported from AirSensor (`main.cpp:210`) — non-blocking, rate-limited (5 s) reconnect attempt from inside `loop()`, called alongside the WiFi check. On connect, (re-)publishes discovery and current state, and subscribes to both the pattern command topic and `<hostname>/cmd`.

### 2. OTA trigger and wait loop

- `mqttCallback()` (ported from AirSensor `main.cpp:361`): on `<hostname>/cmd` payload `"ota"`, sets `s_otaRequested = true`; on `"cancel"`, sets `s_otaCancelled = true`. Also clears the retained command after handling, same as AirSensor.
- `loop()` checks `s_otaRequested` each iteration. When set, calls `enterOTAMode()` instead of running the normal LED/web-server cycle for that pass.
- `enterOTAMode()`:
  - `ArduinoOTA.setHostname(hostname)`, no OTA password (matches AirSensor; relies on trusted LAN + MQTT gating entry into this mode).
  - `ArduinoOTA.begin()`.
  - Blocking loop: `while (!s_otaCancelled && !timedOut) { ArduinoOTA.handle(); mqtt.loop(); }`. LED animation and the raw `WiFiServer` web UI are paused for the duration — same trade-off AirSensor accepts for its OTA window.
  - **Idle timeout: 60 seconds.** If no OTA upload has started within 60 s of entering the wait loop, the function returns on its own (`s_otaRequested` reset to `false`), and `loop()` resumes normal operation. This avoids depending on a `"cancel"` MQTT message arriving to escape OTA-wait — important because this device doesn't deep-sleep and can't rely on the same recovery path AirSensor uses.
  - `ArduinoOTA.onStart/onEnd/onProgress/onError` handlers log via `Serial`, matching AirSensor's handlers.

### 3. WiFi watchdog hardening

- Replace the inline reconnect block currently in `loop()` (`main.cpp:471-483`) with a dedicated `checkWiFi()` function:
  - Rate-limited to run at most every `WIFI_CHECK_INTERVAL` (30 s, unchanged).
  - If `WiFi.status() == WL_CONNECTED`: return immediately.
  - If disconnected: `WiFi.disconnect()`, `WiFi.begin(ssid, pass)`, retry up to 20 × 500 ms (bounded — replaces today's indefinite `while` loop in `Connect_to_Wifi()`).
  - On reconnect: `gWiFiServer.begin()` to restart the web server.
  - On failure: log and return; the next 30 s interval retries. `loop()` keeps driving LEDs in the meantime instead of freezing.
- `setup()` keeps its existing blocking initial connect (`Connect_to_Wifi()`) — there's nothing useful to do before WiFi exists on first boot, so no change needed there.
- `mqtt.loop()` / `reconnectMQTT()` recover naturally once WiFi is restored — no special handling needed for MQTT during a WiFi outage beyond the existing "skip if WiFi not connected" guard AirSensor already uses.

### 4. Cleanup (revised)

- **Correction:** `hoeken/PsychicHttp` is not actually unused. `src/main.cpp:7,532` includes and uses `TemplatePrinter.h`/`TemplatePrinter`, which ships as part of the `PsychicHttp` package — the initial grep for the string `"Psychic"` missed this because the class name doesn't contain it. `PsychicHttp` stays in `lib_deps`; no dependency is removed as part of this work.

### Out of scope

- Migrating the web server from raw `WiFiServer` to a different HTTP library.
- An OTA password.
- Any Home Assistant entities beyond the pattern `select` (no brightness control, no additional sensors).
- Removing `PsychicHttp` (superseded — see Cleanup above).

## Testing

- Manual: verify MQTT discovery entity appears in Home Assistant and correctly reflects/controls `gButtonClicked`.
- Manual: publish `"ota"` to `<hostname>/cmd`, confirm device enters OTA-wait and accepts an `pio run -t upload --upload-port <hostname>.local` update.
- Manual: publish `"ota"` and let it idle — confirm auto-resume to normal LED/web operation after 60 s.
- Manual: disconnect WiFi (e.g. power-cycle the AP) for >30 s, confirm the controller reconnects and resumes without a manual reset, and that LEDs keep animating (not frozen) during a short outage window before reconnect completes.
