# MQTT, Home Assistant Integration, and OTA Updates Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add MQTT connectivity with full Home Assistant discovery for the light pattern, an MQTT-triggered OTA update mode with a safety timeout, and a bounded-retry WiFi watchdog to the Christmas Lights ESP32 controller.

**Architecture:** All changes land in the existing single-file firmware, `src/main.cpp`, following this project's and the sibling `AirSensor` project's convention of one monolithic `main.cpp`. Work proceeds in dependency order: rename/cleanup first, then the WiFi watchdog (self-contained), then MQTT plumbing (connect + discovery + state), then bidirectional pattern control, then the OTA trigger itself, finishing with a full build and manual hardware verification pass.

**Tech Stack:** PlatformIO, Arduino framework, ESP32 (`adafruit_feather_esp32_v2`), FastLED, `PubSubClient` (MQTT), `ArduinoOTA`.

## Global Constraints

- Device hostname: `twinkle-back` (was `BackYard-Xmas-Arduino`); derived `device_id`: `twinkle_back`.
- MQTT broker: reuse `homeassistant.local.solace.org`, user `mosquito` — same broker `AirSensor` (`/Users/edinel/src/Air Quality/AirSensor`) already uses.
- Pattern options (unchanged, from existing `gButtonClicked` values): `off, red, blue, green, yellow, cyan, orange, white, gold, hanukkah, nordic`.
- OTA: no password (matches `AirSensor`), 60-second idle timeout before auto-resuming normal operation.
- WiFi watchdog: check every 30 s (`WIFI_CHECK_INTERVAL`, unchanged), bounded reconnect retry of 20 × 500 ms.
- Keep `hoeken/PsychicHttp` in `lib_deps` — `src/main.cpp:7,532` uses `TemplatePrinter.h`/`TemplatePrinter`, which is part of that package (an earlier grep for the literal string "Psychic" missed this; do not remove the dependency).
- No automated test framework exists in this project (embedded firmware, hardware-dependent). Verification is via `pio run` build success plus the manual hardware checklist in the final task — this replaces unit tests throughout.
- PlatformIO binary: `~/.platformio/penv/bin/pio` (not on `PATH` in this shell).

---

### Task 1: Dependencies, secrets, hostname rename

**Files:**
- Modify: `platformio.ini`
- Modify: `include/arduino-secrets.h`
- Modify: `src/main.cpp:22` (hostname define)

**Interfaces:**
- Produces: `#define hostname "twinkle-back"`, `#define device_id "twinkle_back"` (both in `src/main.cpp`), and `mqttServer`/`mqttUser`/`mqttPass` defines (in `include/arduino-secrets.h`) — all later tasks depend on these names.

- [ ] **Step 1: Update `platformio.ini`** — keep `PsychicHttp` (it provides `TemplatePrinter.h`, used by the existing web server), add `PubSubClient`, and add an OTA upload environment mirroring `AirSensor`'s pattern:

```ini
[env:adafruit_feather_esp32_v2]
platform = espressif32
board = adafruit_feather_esp32_v2
framework = arduino
lib_deps = 
	fastled/FastLED@^3.10.3
	hoeken/PsychicHttp
	knolleary/PubSubClient
monitor_speed = 115200
extra_scripts = pre:tools/generate_strings.py

[env:adafruit_feather_esp32_v2_ota]
extends = env:adafruit_feather_esp32_v2
upload_protocol = espota
upload_port = twinkle-back.local
```

- [ ] **Step 2: Add MQTT credentials to `include/arduino-secrets.h`**

Current content:
```cpp

#define ssid "Treelined"        // your network SSID (name)
#define pass "ShadowNinja"    // your network password (use for WPA, or use as key for WEP)
```

New content:
```cpp

#define ssid "Treelined"        // your network SSID (name)
#define pass "ShadowNinja"    // your network password (use for WPA, or use as key for WEP)
#define mqttServer "homeassistant.local.solace.org"
#define mqttUser "mosquito"
#define mqttPass "334Laidley"
```

- [ ] **Step 3: Rename hostname and add device_id in `src/main.cpp`**

Change `src/main.cpp:22` from:
```cpp
#define hostname "BackYard-Xmas-Arduino"
```
to:
```cpp
#define hostname "twinkle-back"
#define device_id "twinkle_back"
```

- [ ] **Step 4: Verify the build still compiles**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2`
Expected: `SUCCESS` — no references to `PsychicHttp` remain, and the hostname rename doesn't break anything (only used in `WiFi.setHostname(hostname)` at this point).

- [ ] **Step 5: Commit**

```bash
git add platformio.ini include/arduino-secrets.h src/main.cpp
git commit -m "Rename hostname to twinkle-back, add MQTT deps/secrets, drop unused PsychicHttp"
```

---

### Task 2: WiFi watchdog with bounded retry

**Files:**
- Modify: `src/main.cpp:102-115` (`Connect_to_Wifi`, unchanged, still used by `setup()`)
- Modify: `src/main.cpp:471-483` (inline reconnect block in `loop()`, replaced)

**Interfaces:**
- Consumes: `ssid`, `pass` (existing defines), `gWiFiServer` (existing global `WiFiServer`), `gLastWifiCheck` (existing global), `WIFI_CHECK_INTERVAL` (existing const), `debug` (existing global), `Print_Wifi_Status()` (existing function).
- Produces: `void checkWiFi()` — called once per `loop()` iteration by later tasks (task 3 wires MQTT reconnect right after it).

- [ ] **Step 1: Add `checkWiFi()` after `Print_Wifi_Status()`**

Insert after `src/main.cpp:130` (the closing brace of `Print_Wifi_Status()`):

```cpp
void checkWiFi() {
  unsigned long now = millis();
  if (now - gLastWifiCheck < WIFI_CHECK_INTERVAL) return;
  gLastWifiCheck = now;

  if (WiFi.status() == WL_CONNECTED) return;

  Serial.println("WiFi lost, reconnecting...");
  WiFi.disconnect();
  WiFi.begin(ssid, pass);
  for (int i = 0; i < 20 && WiFi.status() != WL_CONNECTED; i++) delay(500);

  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("WiFi restored");
    if (debug) { Print_Wifi_Status(); }
    gWiFiServer.begin();
  } else {
    Serial.println("WiFi reconnect failed, will retry");
  }
}
```

- [ ] **Step 2: Replace the inline reconnect block in `loop()`**

Change (`src/main.cpp:471-483`):
```cpp
void loop() {
  // Periodically check WiFi and reconnect if dropped
  unsigned long now = millis();
  if (now - gLastWifiCheck >= WIFI_CHECK_INTERVAL) {
    gLastWifiCheck = now;
    if (WiFi.status() != WL_CONNECTED) {
      Serial.println("WiFi lost, reconnecting...");
      WiFi.disconnect();
      Connect_to_Wifi();
      gWiFiServer.begin();
      if (debug) { Print_Wifi_Status(); }
    }
  }

  WiFiClient client = gWiFiServer.available();   // Listen for incoming clients
```
to:
```cpp
void loop() {
  checkWiFi();

  WiFiClient client = gWiFiServer.available();   // Listen for incoming clients
```

- [ ] **Step 3: Verify the build compiles**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2`
Expected: `SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add src/main.cpp
git commit -m "Replace unbounded WiFi reconnect with rate-limited, bounded-retry checkWiFi()"
```

---

### Task 3: MQTT connection, Home Assistant discovery, state publish

**Files:**
- Modify: `src/main.cpp` (includes, globals, `setup()`, new functions)

**Interfaces:**
- Consumes: `hostname`, `device_id` (task 1), `mqttServer`/`mqttUser`/`mqttPass` (task 1), `checkWiFi()` (task 2), `gButtonClicked` (existing global `String`).
- Produces: `WiFiClient wifiClient`, `PubSubClient mqtt` (globals later tasks call `mqtt.publish`/`mqtt.subscribe`/`mqtt.loop` on), `char CMD_TOPIC[48]`, `char PATTERN_SET_TOPIC[48]` (task 4 reads these in `mqttCallback`), `void publishPatternState()` (task 4 calls this after a web-triggered pattern change), `bool s_otaRequested`, `bool s_otaCancelled` (task 5 reads/writes these).

- [ ] **Step 1: Add the MQTT include**

At the top of `src/main.cpp`, after `#include <FastLED.h>`:
```cpp
#include <PubSubClient.h>
```

- [ ] **Step 2: Add MQTT globals**

After `#define hostname "twinkle-back"` / `#define device_id "twinkle_back"` (task 1), add:
```cpp
#define MQTT_PORT 1883
#define MQTT_RETRY_INTERVAL_MS 5000
```

After the existing `WiFiServer gWiFiServer(80);` declaration (`src/main.cpp:46`), add:
```cpp
WiFiClient   wifiClient;
PubSubClient mqtt(wifiClient);

char CMD_TOPIC[48];
char PATTERN_STATE_TOPIC[48];
char PATTERN_SET_TOPIC[48];
char PATTERN_CONFIG_TOPIC[64];

bool s_otaRequested = false;
bool s_otaCancelled = false;
```

- [ ] **Step 3: Add a forward-declared, minimal `mqttCallback`, discovery, state-publish, and reconnect functions**

Insert after `checkWiFi()` (end of task 2's addition):

```cpp
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  // Pattern-set and OTA command handling added in later tasks.
}

void publishPatternDiscovery() {
  char payload[640];
  snprintf(payload, sizeof(payload),
    "{\"name\":\"Pattern\","
    "\"state_topic\":\"%s\","
    "\"command_topic\":\"%s\","
    "\"options\":[\"off\",\"red\",\"blue\",\"green\",\"yellow\",\"cyan\",\"orange\",\"white\",\"gold\",\"hanukkah\",\"nordic\"],"
    "\"unique_id\":\"%s_pattern\","
    "\"device\":{\"identifiers\":[\"%s\"],\"name\":\"Twinkle Back\","
    "\"model\":\"ESP32 Christmas Lights\",\"manufacturer\":\"DIY\"}}",
    PATTERN_STATE_TOPIC, PATTERN_SET_TOPIC, device_id, device_id);
  mqtt.publish(PATTERN_CONFIG_TOPIC, payload, true);
  Serial.println("HA discovery published");
}

void publishPatternState() {
  mqtt.publish(PATTERN_STATE_TOPIC, gButtonClicked.c_str(), true);
}

void reconnectMQTT() {
  static unsigned long lastAttempt = 0;
  if (mqtt.connected()) return;
  if (WiFi.status() != WL_CONNECTED) return;

  unsigned long now = millis();
  if (now - lastAttempt < MQTT_RETRY_INTERVAL_MS) return;
  lastAttempt = now;

  Serial.print("Connecting to MQTT: ");
  Serial.println(mqttServer);

  if (mqtt.connect(hostname, mqttUser, mqttPass)) {
    Serial.println("MQTT connected");
    publishPatternDiscovery();
    publishPatternState();
    mqtt.subscribe(PATTERN_SET_TOPIC);
    mqtt.subscribe(CMD_TOPIC);
  } else {
    Serial.print("MQTT connect failed, rc=");
    Serial.println(mqtt.state());
  }
}
```

- [ ] **Step 4: Build MQTT topics and configure the client in `setup()`**

In `setup()`, immediately after `Connect_to_Wifi();` (`src/main.cpp:461`), add:
```cpp
  snprintf(CMD_TOPIC,           sizeof(CMD_TOPIC),           "%s/cmd",                          hostname);
  snprintf(PATTERN_STATE_TOPIC, sizeof(PATTERN_STATE_TOPIC), "%s/select/pattern/state",          hostname);
  snprintf(PATTERN_SET_TOPIC,   sizeof(PATTERN_SET_TOPIC),   "%s/select/pattern/set",             hostname);
  snprintf(PATTERN_CONFIG_TOPIC, sizeof(PATTERN_CONFIG_TOPIC), "homeassistant/select/%s/pattern/config", device_id);

  mqtt.setServer(mqttServer, MQTT_PORT);
  mqtt.setBufferSize(768);
  mqtt.setCallback(mqttCallback);
```

- [ ] **Step 5: Call `reconnectMQTT()` and `mqtt.loop()` from `loop()`**

Change the top of `loop()` from:
```cpp
void loop() {
  checkWiFi();

  WiFiClient client = gWiFiServer.available();   // Listen for incoming clients
```
to:
```cpp
void loop() {
  checkWiFi();
  reconnectMQTT();
  mqtt.loop();

  WiFiClient client = gWiFiServer.available();   // Listen for incoming clients
```

- [ ] **Step 6: Verify the build compiles**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2`
Expected: `SUCCESS`

- [ ] **Step 7: Commit**

```bash
git add src/main.cpp
git commit -m "Add MQTT connection, HA discovery, and pattern state publishing"
```

---

### Task 4: Bidirectional pattern control (HA → device, device → HA)

**Files:**
- Modify: `src/main.cpp` (`mqttCallback`, the web-request color-handling block in `loop()`)

**Interfaces:**
- Consumes: `PATTERN_SET_TOPIC`, `CMD_TOPIC`, `publishPatternState()`, `mqtt` (all from task 3), `gButtonClicked` (existing global), `s_otaRequested`/`s_otaCancelled` (task 3 globals — OTA branch is wired here but acted on starting task 5).

- [ ] **Step 1: Implement `mqttCallback`**

Replace the stub body from task 3:
```cpp
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  // Pattern-set and OTA command handling added in later tasks.
}
```
with:
```cpp
void mqttCallback(char* topic, byte* payload, unsigned int length) {
  if (strcmp(topic, PATTERN_SET_TOPIC) == 0) {
    char value[16] = {};
    unsigned int len = length < sizeof(value) - 1 ? length : sizeof(value) - 1;
    memcpy(value, payload, len);
    gButtonClicked = String(value);
    Serial.print("Pattern set via MQTT: ");
    Serial.println(gButtonClicked);
    publishPatternState();
  } else if (strcmp(topic, CMD_TOPIC) == 0) {
    if (length == 3 && strncmp((char*)payload, "ota", 3) == 0) {
      Serial.println("OTA requested via MQTT");
      s_otaRequested = true;
      mqtt.publish(CMD_TOPIC, "", true);  // clear the retained command
    } else if (length == 6 && strncmp((char*)payload, "cancel", 6) == 0) {
      Serial.println("OTA cancelled via MQTT");
      s_otaCancelled = true;
      mqtt.publish(CMD_TOPIC, "", true);  // clear the retained command
    }
  }
}
```

- [ ] **Step 2: Publish state after a web-triggered pattern change**

In `loop()`'s HTTP request handling, find the end of the color `else if` chain (`src/main.cpp:507-529`, ending with the `"nordic"` branch) and the line immediately after it that reads:
```cpp
            }
            // Display the HTML web page, using the TemplatePrinter to make it a little easier to deal with.
```
Insert `publishPatternState();` between the closing `}` of the if-chain and the comment:
```cpp
            }
            publishPatternState();
            // Display the HTML web page, using the TemplatePrinter to make it a little easier to deal with.
```

- [ ] **Step 3: Verify the build compiles**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2`
Expected: `SUCCESS`

- [ ] **Step 4: Commit**

```bash
git add src/main.cpp
git commit -m "Wire bidirectional pattern control between web UI and Home Assistant"
```

---

### Task 5: OTA trigger with 60-second idle timeout

**Files:**
- Modify: `src/main.cpp` (includes, globals, new `enterOTAMode()`, `loop()`)

**Interfaces:**
- Consumes: `hostname` (task 1), `s_otaRequested`/`s_otaCancelled` (task 3), `mqtt` (task 3).
- Produces: `void enterOTAMode()` — called from `loop()`, no other task depends on it.

- [ ] **Step 1: Add the ArduinoOTA include**

After `#include <PubSubClient.h>` (task 3), add:
```cpp
#include <ArduinoOTA.h>
```

- [ ] **Step 2: Add the idle-timeout constant and upload-tracking flag**

After `#define MQTT_RETRY_INTERVAL_MS 5000` (task 3), add:
```cpp
#define OTA_IDLE_TIMEOUT_MS 60000
```

After `bool s_otaCancelled = false;` (task 3), add:
```cpp
bool s_otaUploadStarted = false;
```

- [ ] **Step 3: Implement `enterOTAMode()`**

Insert after `reconnectMQTT()` (end of task 3's block):
```cpp
void enterOTAMode() {
  Serial.println("OTA mode — waiting for upload");
  s_otaUploadStarted = false;

  ArduinoOTA.setHostname(hostname);
  ArduinoOTA.onStart([]()  { s_otaUploadStarted = true; Serial.println("OTA upload started"); });
  ArduinoOTA.onEnd([]()    { Serial.println("OTA complete — rebooting"); });
  ArduinoOTA.onProgress([](unsigned int prog, unsigned int total) {
    Serial.printf("OTA %u%%\n", prog * 100 / total);
  });
  ArduinoOTA.onError([](ota_error_t e) { Serial.printf("OTA error %u\n", e); });
  ArduinoOTA.begin();

  unsigned long start = millis();
  while (!s_otaCancelled) {
    ArduinoOTA.handle();
    mqtt.loop();
    if (!s_otaUploadStarted && millis() - start > OTA_IDLE_TIMEOUT_MS) {
      Serial.println("OTA idle timeout — resuming normal operation");
      break;
    }
  }

  s_otaRequested  = false;
  s_otaCancelled  = false;
  Serial.println("Exiting OTA mode");
}
```

- [ ] **Step 4: Wire OTA entry into `loop()`**

Change the top of `loop()` from:
```cpp
void loop() {
  checkWiFi();
  reconnectMQTT();
  mqtt.loop();

  WiFiClient client = gWiFiServer.available();   // Listen for incoming clients
```
to:
```cpp
void loop() {
  checkWiFi();
  reconnectMQTT();
  mqtt.loop();

  if (s_otaRequested) {
    enterOTAMode();
    return;
  }

  WiFiClient client = gWiFiServer.available();   // Listen for incoming clients
```

- [ ] **Step 5: Verify the build compiles**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2`
Expected: `SUCCESS`

- [ ] **Step 6: Commit**

```bash
git add src/main.cpp
git commit -m "Add MQTT-triggered OTA mode with 60s idle timeout"
```

---

### Task 6: Full build and manual hardware verification

**Files:** none (verification only)

- [ ] **Step 1: Full clean build**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2 -t clean && ~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2`
Expected: `SUCCESS`, no warnings about undefined references.

- [ ] **Step 2: Flash over USB and confirm baseline operation**

Run: `~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2 -t upload`
Then open the serial monitor (`~/.platformio/penv/bin/pio device monitor -b 115200`) and confirm:
- WiFi connects and prints an IP.
- `"MQTT connected"` and `"HA discovery published"` appear.
- The web UI at the printed IP still loads and changing a color updates the LEDs (unchanged existing behavior).

- [ ] **Step 3: Verify Home Assistant discovery**

In Home Assistant, confirm a new entity (`select.twinkle_back_pattern` or similar, device "Twinkle Back") appears, its current state matches `gButtonClicked`, and selecting a different option in HA changes the physical LED pattern.

- [ ] **Step 4: Verify web → HA sync**

Change the pattern from the device's own web UI; confirm the Home Assistant entity's state updates to match within a few seconds.

- [ ] **Step 5: Verify OTA trigger and successful upload**

Publish `"ota"` to `twinkle-back/cmd` (e.g. via Home Assistant MQTT developer tools or `mosquitto_pub -h homeassistant.local.solace.org -u mosquito -P 334Laidley -t twinkle-back/cmd -m ota`).
Confirm the serial log shows `"OTA mode — waiting for upload"`, then run:
`~/.platformio/penv/bin/pio run -e adafruit_feather_esp32_v2_ota -t upload`
Confirm the upload completes and the device reboots running the new firmware.

- [ ] **Step 6: Verify OTA idle timeout**

Publish `"ota"` again but do not start an upload. After 60 seconds, confirm the serial log shows `"OTA idle timeout — resuming normal operation"` and that the LED animation and web UI resume without a manual reset.

- [ ] **Step 7: Verify WiFi watchdog**

With the device running, disable the WiFi access point (or block the device) for at least 30 seconds, then restore it. Confirm via serial log that `checkWiFi()` detects the drop, reconnects within the 20×500ms retry window once the AP is back, and that LED animation continues throughout (not frozen) except during the brief reconnect attempt itself.

- [ ] **Step 8: Final commit (if any fixes were needed during verification)**

If steps 1-7 required any code fixes, stage and commit them:
```bash
git add src/main.cpp platformio.ini include/arduino-secrets.h
git commit -m "Fix issues found during MQTT/OTA hardware verification"
```
If no fixes were needed, no commit is required for this task.
