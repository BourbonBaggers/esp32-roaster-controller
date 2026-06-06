# Boozebaggers Wood Roaster Controller — Architectural Redesign

> **Living document:** This plan is stored at `docs/architecture.md` inside the project repository so it remains accessible without Claude. The copy at `/Users/jaykreusch/.claude/plans/` is the working draft — they should stay in sync.

## Context

The current system is a monolithic 1089-line Arduino sketch that runs a web server directly on the ESP32. This creates a poor separation of concerns: the microcontroller serves HTML, runs a PID loop, and sends IFTTT webhooks all from one file. There is no data persistence, no historical analysis, and no way to dial in roast consistency over time.

The redesign decouples the ESP32 into a lean IoT data publisher and actuator, while a Docker container on the local Ubuntu VM becomes the brain: storing every roast to SQLite, serving a proper web dashboard, firing Klaviyo alerts, and managing wood-type profiles with timed procedure reminders.

The business need is clear: Jay roasts 4 wood types for Boozebaggers (toasted oak, charred oak, smoke infusion, toasted cherry) and wants to build data over time to correlate temperature/duration with output color — achieving batch consistency.

---

## Architecture Overview

```
ESP32 Feather v2 (LAN)
  ├── Reads AD8495 temp every 2s
  ├── Drives stepper motor (heat control)
  ├── Runs PID loop locally (SAFETY: works if server down)
  ├── Publishes → MQTT roaster/telemetry (JSON, every 2s)
  └── Subscribes ← MQTT roaster/commands

Docker on Ubuntu VM (same LAN, always-on Proxmox VM)
  ├── mosquitto  — MQTT broker (eclipse-mosquitto:2)
  └── app        — FastAPI (Python 3.12)
        ├── Subscribes to MQTT telemetry → SQLite
        ├── Broadcasts via WebSocket → browser
        ├── REST API for commands, sessions, profiles, history
        ├── Klaviyo v3 Track API for alerts
        ├── Profile alert scheduler (background asyncio task)
        └── Serves static HTML/JS dashboard

Browser (phone/tablet/laptop on LAN)
  └── WebSocket → live dashboard, historical charts, session management
```

**Why MQTT over HTTP POST:** MQTT is the de facto standard protocol for IoT devices — it's an ISO/IEC standard (20922), used by AWS IoT Core, Azure IoT Hub, Google Cloud IoT, and Home Assistant. Eclipse Mosquitto (the broker we use) is the reference open-source implementation maintained by the Eclipse Foundation, with enormous community support and documentation. This is as mainstream as it gets for device-to-server communication. The broker decouples the ESP32 from the server — if the server restarts, messages queue. The ESP32 handles reconnection natively. The Docker Compose stack gains one tiny service (5MB image) for a major gain in robustness and clean bidirectional communication. `paho-mqtt` is the official Python MQTT client library, also Eclipse-maintained.

**Why PID stays on device:** A wood roaster running without temperature control during a network hiccup is a fire hazard. The ESP32 must be able to maintain its last-known setpoint and PID loop independently of the server.

---

## Project File Structure

```
/Users/jaykreusch/Documents/ESP32 Roaster Controller/
├── docs/
│   └── architecture.md               (copy of this plan — lives in the repo)
├── firmware/
│   ├── platformio.ini
│   └── src/
│       └── RoasterPIDv2.ino          (~300 lines, replaces 1089-line monolith)
└── docker/
    ├── docker-compose.yml
    ├── mosquitto/
    │   └── mosquitto.conf
    ├── app/
    │   ├── Dockerfile
    │   ├── requirements.txt
    │   ├── main.py                   (FastAPI app + lifespan + WebSocket)
    │   ├── mqtt_handler.py           (paho-mqtt → asyncio bridge — critical)
    │   ├── database.py               (aiosqlite + ROR calc + profile seeding)
    │   ├── klaviyo_handler.py        (httpx → Klaviyo v3 Track API)
    │   ├── profile_scheduler.py      (background alert timing task)
    │   ├── models.py                 (Pydantic request/response models)
    │   └── static/
    │       ├── index.html
    │       └── app.js
    └── simulator/
        ├── Dockerfile
        ├── requirements.txt
        └── simulator.py              (software ESP32 stand-in for dev/testing)
```

---

## Auto-Discovery (mDNS / Bonjour)

Rather than hardcoding IPs, the ESP32 resolves the broker by name using **mDNS** (multicast DNS — the same protocol your Mac uses for `.local` hostnames, and how printers/Chromecasts discover your network). This means if your subnet changes or you swap network hardware, nothing needs to be recoded.

**How it works:**
- The Ubuntu VM advertises itself as `roaster-hub.local` via `avahi-daemon` (Linux's mDNS implementation, extremely well-supported and package-available via `apt`)
- The ESP32 uses the `ESPmDNS` library (already built into the ESP32 Arduino core — no extra library needed) to resolve `roaster-hub.local` → IP at startup
- If resolution fails (first boot, network hiccup), the ESP32 retries every 10 seconds before falling back to a hardcoded config fallback IP

**Firmware change:**
```cpp
#include <ESPmDNS.h>

// Replace hardcoded IP with:
const char* mqttBrokerHostname = "roaster-hub.local";

// In connectWifi(), after WiFi connects:
if (!MDNS.begin("roaster-esp32")) { ... }
IPAddress brokerIP = MDNS.queryHost(mqttBrokerHostname);
// brokerIP is passed to mqttClient.connect() instead of a string hostname
```

**Docker side — `avahi-daemon` on Ubuntu VM:**
Add to `docker-compose.yml`: the `app` service uses `network_mode: host` OR the Ubuntu VM itself runs `avahi-daemon` with a service file. Since this is an always-on VM (not a container), install `avahi-daemon` on the Ubuntu host directly:
```bash
sudo apt install avahi-daemon
sudo systemctl enable avahi-daemon
```
Then create `/etc/avahi/services/roaster.service`:
```xml
<?xml version="1.0" standalone='no'?>
<service-group>
  <name>Roaster Hub</name>
  <service>
    <type>_mqtt._tcp</type>
    <port>1883</port>
  </service>
</service-group>
```
This advertises the MQTT service on the LAN so any mDNS-aware client can discover it. The hostname `roaster-hub.local` is set in `/etc/hostname` and `/etc/avahi/avahi-daemon.conf`.

**Belt-and-suspenders approach:** Use both static DHCP reservation in your router AND mDNS. Static reservation ensures the IP stays consistent even if mDNS is temporarily unavailable. mDNS ensures you never need to touch firmware if the IP ever changes.

---

## Device Simulator (`docker/simulator/simulator.py`)

A Python script that impersonates the ESP32 on MQTT — publishes realistic telemetry and responds to commands. This lets you build, test, and iterate on the entire Docker stack and dashboard without the physical hardware connected.

**Usage:**
```bash
# From docker/ directory:
docker compose --profile sim up     # starts broker + app + simulator together

# Or run standalone:
cd docker/simulator
python simulator.py --profile "Toasted Oak" --start-temp 72 --speed 10
```

`--speed 10` runs simulation at 10× real time so you can test a 45-minute roast in 4.5 minutes.

**What the simulator does:**
- Connects to `mosquitto:1883` (same broker as the real ESP32 would)
- Publishes to `roaster/telemetry` every 2 seconds (or faster in `--speed` mode)
- Subscribes to `roaster/commands` and responds exactly as the firmware would:
  - `setTemp` → updates simulated target
  - `moveStepper` → nudges simulated temperature (CW = +random 1-5°F, CCW = −random 1-5°F)
  - `setMode` → toggles simulated auto mode
  - `setPid` → updates simulated PID parameters
  - `resetNetSteps`, `resetElapsed`, `setSimTemp` → updates internal state
- Simulates a realistic temperature curve:
  - Heats toward `setTemp` at a rate influenced by net stepper position
  - Adds small random noise (±1°F) to mimic real sensor readings
  - Simulates thermal inertia (doesn't snap to target instantly)
- Prints a running log to stdout: `[12s] Temp: 243.2°F → 450.0°F, ROR: +14.2°F/min, Steps: +8`

**Temperature physics model (simple but realistic):**
```python
# Each 2s tick:
heat_rate = 0.8  # baseline degrees per second when at full heat
cool_rate = 0.3  # passive heat loss
stepper_effect = net_steps * 0.15  # more CW steps = more heat input
delta = (heat_rate + stepper_effect - cool_rate) * 2  # 2s tick
# Add thermal lag: temp moves toward target_temp asymptotically
temp += delta * (1 - abs(current_temp - target_temp) / 500)
```

**`docker-compose.yml` simulator service:**
```yaml
simulator:
  build: ./simulator
  container_name: roaster-simulator
  environment:
    - MQTT_BROKER=mosquitto
    - MQTT_PORT=1883
    - SIM_PROFILE=Toasted Oak
    - SIM_SPEED=1
  depends_on:
    - mosquitto
  profiles:
    - sim   # only starts with: docker compose --profile sim up
```

The `profiles: [sim]` key means the simulator does NOT start during normal `docker compose up` — it only runs when you explicitly pass `--profile sim`. This prevents accidental simulator data flooding your real roast history.

---

## MQTT Contract

### `roaster/telemetry` (ESP32 → server, every 2s, QoS 0)
```json
{ "temp": 427.3, "setTemp": 450.0, "netSteps": 42,
  "pidOutput": 3.7, "isAuto": true, "uptime": 1823, "simMode": false }
```

### `roaster/commands` (server → ESP32, QoS 1)
```json
{ "cmd": "setTemp",     "value": 455.0 }
{ "cmd": "setMode",     "auto": true }
{ "cmd": "moveStepper", "dir": "CW", "steps": 5 }
{ "cmd": "setPid",      "kp": 5.0, "ki": 0.5, "kd": 2.0 }
{ "cmd": "setSimMode",  "enabled": true }
{ "cmd": "resetNetSteps" }
{ "cmd": "resetElapsed" }
{ "cmd": "setSimTemp",  "temp": 300.0 }
```

**Important:** `setTemp` sends the absolute new target (not a delta). The firmware assigns `setTemp = doc["value"]` directly, not `setTemp += amount`.

---

## Database Schema (SQLite, Postgres-migration-compatible)

```sql
roast_profiles (
  id INTEGER PRIMARY KEY, name TEXT NOT NULL,
  wood_type TEXT, target_temp REAL, roast_duration_min INTEGER,
  post_roast_procedure TEXT,  -- JSON array of {step}
  alert_schedule TEXT,        -- JSON array of {offset_min, message, event_type}
  notes TEXT, created_at TEXT
)

roast_sessions (
  id INTEGER PRIMARY KEY, profile_id INTEGER REFERENCES roast_profiles(id),
  started_at TEXT NOT NULL, ended_at TEXT,
  batch_weight_lbs REAL, peak_temp REAL,
  color_rating TEXT, color_notes TEXT, operator_notes TEXT,
  klaviyo_profile_id TEXT, status TEXT  -- "active"|"completed"|"aborted"
)

roast_telemetry (
  id INTEGER PRIMARY KEY, session_id INTEGER REFERENCES roast_sessions(id),
  recorded_at TEXT NOT NULL, temperature REAL NOT NULL,
  set_temp REAL, net_steps INTEGER, pid_output REAL,
  ror_per_min REAL,          -- calculated server-side (60s rolling window)
  is_auto_mode INTEGER
)

roast_events (
  id INTEGER PRIMARY KEY, session_id INTEGER REFERENCES roast_sessions(id),
  occurred_at TEXT NOT NULL, temperature REAL,
  event_type TEXT,           -- "cut_flame"|"eject"|"lid_on"|"cool_start"|"alert_sent"|"custom"
  message TEXT, sent_to_klaviyo INTEGER
)
```

Enable WAL mode on init: `PRAGMA journal_mode=WAL` to prevent read/write contention at 2s insert rate.

---

## Firmware Changes (`firmware/src/RoasterPIDv2.ino`)

**Remove:** `handleRoot` and all 640-line embedded HTML, `handleMoveStepper`, `handleStatus`, all `server.on(...)` handlers, `WebServer`, `HTTPClient`, `sendIftttAlert`, `serialBuffer`, `escapeJson`, `WebServer.h`.

**Keep:** `readTemperature` (exact formula preserved: `(voltage - 1.25) / 0.005` → C → F), `moveStepper` (microstepping ×4 preserved), PID setup/compute, `updateOLED`, `connectWifi`, `logToSerial`, `formatElapsedTime`.

**Add:**
- `PubSubClient` (MQTT client) + `ArduinoJson` v7 + `ESPmDNS` (built-in) includes
- `mqttReconnect()` — non-blocking, 5s cooldown, uses mDNS-resolved IP, subscribes to `roaster/commands` on connect
- `resolveBroker()` — called once after WiFi connects; uses `MDNS.queryHost("roaster-hub.local")` to get broker IP; falls back to configurable fallback IP if mDNS fails
- `publishTelemetry()` — builds JSON with `JsonDocument`, publishes every 2s
- `mqttCallback()` — dispatches on `cmd` field, applies `setTemp` as absolute assignment
- OLED shows MQTT connection status ("MQTT: OK" / "MQTT: --") alongside temp

**`platformio.ini` lib_deps to add:**
```
knolleary/PubSubClient @ ^2.8
adafruit/Adafruit SSD1306 @ ^2.5.7
adafruit/Adafruit GFX Library @ ^1.11.9
br3ttb/Arduino PID Library @ ^1.2.1
bblanchon/ArduinoJson @ 7.0.4
```
Also add: `build_flags = -DMQTT_MAX_PACKET_SIZE=512`

---

## Docker Services

### `docker-compose.yml`
- `mosquitto`: ports 1883 (MQTT) + 9001 (WebSocket, for future debug). `allow_anonymous true` for LAN-only.
- `app`: FastAPI + uvicorn, port 8000. Volumes: `./app:/app` (live reload during dev) + named `roaster-db:/data` for SQLite persistence. Env vars: `MQTT_BROKER`, `DB_PATH`, `KLAVIYO_API_KEY`, `KLAVIYO_PROFILE_EMAIL`.

### Key Python Library Choices
- `paho-mqtt==2.0.0` — new callback API (5-arg `on_connect`), use `CallbackAPIVersion.VERSION2`
- `aiosqlite==0.20.0` — async SQLite, never blocks FastAPI event loop
- `httpx==0.27.0` — async HTTP for Klaviyo calls
- `fastapi==0.111.0` + `uvicorn[standard]`

---

## FastAPI Key Design Points

### Thread Bridge (most critical file: `mqtt_handler.py`)
`paho-mqtt` runs in a daemon thread via `loop_start()`. To write to SQLite and push WebSocket without race conditions, use:
```python
asyncio.run_coroutine_threadsafe(_dispatch(data), _loop)
```
Capture `_loop` inside the `lifespan` coroutine via `asyncio.get_event_loop()`. Never capture it at module import time.

### WebSocket Push
Maintain a `set[WebSocket]` of active connections. The `on_telemetry` callback: writes to DB → calculates ROR → broadcasts `{"type":"telemetry","data":{...}}` to all clients. Dead connections are pruned on send failure.

### REST Endpoints
```
POST /api/sessions/start          — select profile, enter batch weight/notes
POST /api/sessions/{id}/end       — enter color rating + notes
GET  /api/sessions                — list (50 most recent)
GET  /api/sessions/{id}/telemetry — time-series for historical chart
GET/POST/DELETE /api/profiles     — profile CRUD
POST /api/commands                — validates CommandPayload → MQTT publish
GET  /api/status                  — latest telemetry snapshot
POST /api/events                  — manually log a roast event
```

### ROR Calculation (in `database.py`)
Fetch the reading from 60 seconds ago for the active session, compute `delta_temp / 1.0 minute`. The 60-second lookback window gives a stable, smooth ROR. Do not use consecutive 2-second readings — too noisy.

---

## Klaviyo Integration (`klaviyo_handler.py`)

Use Klaviyo v3 Track API: `POST https://a.klaviyo.com/api/events/`  
Headers: `Authorization: Klaviyo-API-Key {pk_...}`, `revision: 2024-02-15`

**One event, one flow.** All notifications use a single Klaviyo event name — `Roast Notification` — so only one flow and one trigger need to be configured in Klaviyo. The `alert_type` property drives conditional content blocks inside that flow to vary the subject line and body per notification type.

**Event name:** `Roast Notification`

**`alert_type` values and when they fire:**
| `alert_type` | When |
|---|---|
| `temp_alert` | Temp deviates > threshold from setpoint for > 30s |
| `progress_update` | Every 5 minutes during active session (replaces IFTTT) |
| `profile_step` | Profile-scheduled offset reached (cut flame, eject, lid on, etc.) |

**Properties always included:** `session_id`, `wood_type`, `current_temp`, `elapsed_min`, `alert_type`, `message`, `subject`.

`subject` is included in the event payload so Klaviyo's conditional blocks can set the email subject line without needing separate flows.

---

## Profile Scheduler (`profile_scheduler.py`)

Background `asyncio` task, checks every 15 seconds. For the active session, compares `elapsed_min` against each alert's `offset_min` in the profile's `alert_schedule`. Uses an in-memory `set` of `(session_id, offset_min)` to prevent duplicate fires. On app restart, re-checks against `roast_events` table to skip already-fired alerts.

---

## Predefined Profiles (seeded on first DB init)

| Profile | Target | Duration | Key Alerts |
|---|---|---|---|
| Toasted Oak | 450°F | 45 min | 35m: "10 min until flame cut", 45m: "Cut flame", 75m: "Check cooling" |
| Charred Oak | 450°F | 75 min | 65m: "10 min until flame cut", 75m: "Cut flame", 105m: "Check cooling" |
| Smoke Infusion | 450°F | 75 min | 65m: "10 min until eject", 75m: "Eject", 80m: "Lid on", 135m: "Check smoldering", 195m: "Begin cool" |
| Toasted Cherry | 450°F | 45 min | TBD — editable in UI |

---

## Frontend Features (`static/index.html` + `static/app.js`)

**Live Dashboard:**
- Large current temp display + ROR (°F/min)
- Set temp with +5/-5 buttons
- Manual/Auto toggle, Real/Sim toggle
- CW/CCW stepper buttons with step size slider
- Elapsed time + net steps
- Dual-axis Chart.js: temp (left) + ROR (right, dashed), `animation: false` for live updates
- WebSocket with 3s auto-reconnect

**Session Management:**
- "Start Roast" button → modal: select profile, enter batch weight (lbs) + notes
- "End Roast" button → modal: enter color rating + color notes
- Active session shown in header with elapsed time

**History Tab:**
- Table of past sessions: date, profile, peak temp, color rating, status
- Click row → loads historical chart (x-axis = elapsed minutes, not wall clock)
- Overlay toggle: select a past roast to overlay on any chart for visual comparison

**Color Consistency Table:**
- Filters by wood type / profile
- Shows: date, peak temp, color rating, notes — the QA tool for dialing in batch consistency

**PID Tuning Panel** (`<details>` collapsed by default):
- Editable Kp, Ki, Kd inputs → sends `setPid` command to ESP32

---

## Build Order (testability-first)

1. **`docs/architecture.md`** — copy this plan into the repo as the living reference doc
2. **Ubuntu VM setup** — install `avahi-daemon`, set hostname to `roaster-hub`, confirm `ping roaster-hub.local` works from your Mac
3. **`firmware/platformio.ini`** — add lib_deps (PubSubClient, ArduinoJson, ESPmDNS already built-in)
4. **`firmware/src/RoasterPIDv2.ino`** — rewrite with MQTT + mDNS; test with `mosquitto_sub -t roaster/telemetry` on a standalone broker first
5. **`docker/mosquitto/` + `docker-compose.yml`** — bring up broker only, confirm ESP32 connects to `roaster-hub.local:1883`
6. **`docker/simulator/simulator.py`** — build and start simulator; confirm telemetry flows before writing any backend logic
7. **`docker/app/database.py`** — schema + seeding; test standalone with a Python script
8. **`docker/app/mqtt_handler.py`** — test thread bridge; simulator provides the MQTT traffic
9. **`docker/app/main.py` skeleton + `models.py`** — test REST with curl, simulator running, no frontend yet
10. **`docker/app/klaviyo_handler.py`** — fire a test event to confirm API key + revision header work
11. **`docker/app/profile_scheduler.py`** — test by back-dating `started_at` in DB, simulator running at `--speed 10`
12. **`docker/app/static/`** — build full dashboard; entire flow testable with `docker compose --profile sim up`
13. **`docker/app/Dockerfile` + `requirements.txt`** — finalize container build
14. **End-to-end hardware test** — flash final firmware, run real roast in sim mode first to validate, then switch to real sensor

---

## Risks to Watch

- **paho-mqtt thread bridge**: Capture `asyncio.get_event_loop()` inside `lifespan`, never at import time. Any "attached to a different loop" error traces back here.
- **ESP32 MQTT reconnect**: Verify `PubSubClient.loop()` is non-blocking when broker is unreachable — it is by design, but test with broker stopped to confirm PID keeps running.
- **`setTemp` command semantics**: New command sends absolute value; old handler used a delta. Must not mix these up in firmware or the setpoint will drift unexpectedly.
- **ArduinoJson v6 vs v7 API**: Pin to `bblanchon/ArduinoJson @ 7.0.4`. v7 uses `JsonDocument` not `StaticJsonDocument`.
- **Klaviyo API revision**: `2024-02-15` is the targeted revision. Verify it hasn't been deprecated at build time; check `https://developers.klaviyo.com/en/reference/api_overview`.

---

## Verification Plan

1. Flash firmware → OLED shows "MQTT: OK" → `mosquitto_sub` shows telemetry JSON every 2s
2. `docker compose up` → `http://[vm-ip]:8000` loads dashboard → live temp chart updates
3. Press CW/CCW on dashboard → ESP32 blue LED blinks → stepper moves
4. Start a session with "Toasted Oak" profile → set `simMode: true` → advance time 35 min in DB → confirm Klaviyo event fires
5. End session → fill in color rating → verify in history tab and color consistency table
6. Check SQLite: `sqlite3 /data/roaster.db "SELECT * FROM roast_telemetry LIMIT 5"`
7. Load historical session → overlay a second session → confirm dual curves on chart
