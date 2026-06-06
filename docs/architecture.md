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

# Anomaly modes:
python simulator.py --scenario runaway          # temp climbs past setpoint uncontrolled
python simulator.py --scenario mqtt-drop        # MQTT disconnects mid-roast for 5 minutes
python simulator.py --scenario stuck-valve      # valve stops responding (no temp change on steps)
python simulator.py --scenario slow-heat        # cold/windy day — 30-min warmup curve
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
  - **Warmup: 15–30 minutes from ambient to 450°F** (real-world calibrated range depending on wind/conditions)
  - Heats toward `setTemp` at a rate influenced by net stepper position
  - Adds small random noise (±1°F) to mimic real sensor readings
  - Simulates thermal inertia (doesn't snap to target instantly)
- Prints a running log to stdout: `[12s] Temp: 243.2°F → 450.0°F, ROR: +14.2°F/min, Steps: +8`

**Temperature physics model (calibrated to real hardware):**
```python
# Each 2s tick:
# Real warmup: ~15-30 min from 72°F ambient to 450°F setpoint
# heat_rate tuned so default sim hits 450°F in ~20 min at speed=1
heat_rate = 0.35   # degrees per second at reference step position
cool_rate = 0.08   # passive heat loss (ambient ~72°F)
stepper_effect = net_steps * 0.06  # more CW steps = more heat input
delta = (heat_rate + stepper_effect - cool_rate) * 2  # 2s tick
# Thermal lag: asymptotic approach to target
temp += delta * (1 - abs(current_temp - target_temp) / 600)

# Noise model: real environment is NOT a lab. Normal: ±1-2°F every 2s.
# Windy/cold conditions: ±5-10°F swings. Modulated by wind_speed parameter.
noise = random.gauss(0, noise_sigma)   # noise_sigma driven by wind_speed
temp += noise

# Blowout detection threshold: 30°F drop in a short window = likely blowout
# Simulator can inject a blowout by cutting heat_rate to 0 for one tick

# Anomaly scenarios override normal physics:
# runaway:    stepper_effect ignored, heat_rate *= 3
# stuck-valve: stepper_effect = 0 always (valve not responding)
# mqtt-drop:  client.disconnect() at t=5min, reconnect at t=10min
# slow-heat:  heat_rate *= 0.5, wind_speed = 20 (cold/windy day)
# blowout:    heat_rate = 0 for 30s, then normal (simulates flame out then relight)
```

**Weather simulation parameters** (all configurable as CLI flags or env vars):
```bash
python simulator.py \
  --ambient-temp 65      # starting ambient temp (°F)
  --wind-speed 8         # mph; drives noise_sigma (calm=1°F σ, 20mph=8°F σ)
  --wind-variability 0.3 # how much wind gusts (0=steady, 1=very gusty)
  --scenario blowout     # inject a blowout at t=15min
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
  lot_id TEXT NOT NULL,       -- auto-assigned sequential (e.g. "LOT-0042"), user may override with custom label
  started_at TEXT NOT NULL, ended_at TEXT,
  batch_volume_gal REAL,      -- standard batch = 5 gallons; recorded for reference
  ambient_temp_f REAL,        -- fetched from Open-Meteo at session start
  wind_speed_mph REAL,        -- fetched from Open-Meteo at session start
  weather_fetched_at TEXT,    -- ISO timestamp of weather fetch
  peak_temp REAL,
  color_rating TEXT, color_notes TEXT,
  batch_photo_path TEXT,      -- optional JPEG upload for this batch's color
  operator_notes TEXT,
  klaviyo_profile_id TEXT,
  status TEXT                 -- "active"|"completed"|"aborted"|"emergency"
)

roast_telemetry (
  id INTEGER PRIMARY KEY, session_id INTEGER REFERENCES roast_sessions(id),
  recorded_at TEXT NOT NULL, temperature REAL NOT NULL,
  set_temp REAL, net_steps INTEGER, pid_output REAL,
  ror_per_min REAL,           -- calculated server-side (60s rolling window)
  is_auto_mode INTEGER
)

roast_events (
  id INTEGER PRIMARY KEY, session_id INTEGER REFERENCES roast_sessions(id),
  occurred_at TEXT NOT NULL, temperature REAL,
  event_type TEXT,            -- "cut_flame"|"eject"|"lid_on"|"cool_start"|"alert_sent"|"emergency_close"|"cooldown_ready"|"custom"
  message TEXT, sent_to_klaviyo INTEGER
)

color_reference_photos (
  id INTEGER PRIMARY KEY,
  wood_type TEXT NOT NULL,    -- "toasted_oak"|"charred_oak"|"smoke_infusion"|"toasted_cherry"
  color_label TEXT NOT NULL,  -- e.g. "Light", "Medium", "Dark", "Extra Dark"
  photo_path TEXT NOT NULL,   -- path to JPEG in server's static/color-refs/ directory
  notes TEXT, created_at TEXT
)
```

**Lot ID format:** `{product_code}-{sku}-{YYYYMMDD}-{batch_letter}`

| Product | Code |
|---|---|
| Toasted Oak | `TO` |
| Charred Oak | `CO` |
| Smoke Infusion | `SI` |
| Toasted Cherry | `TC` |
| Cherry Cinnamon | `CC` |

SKU defaults to `BULK`. Batch letter increments A→B→C per day per product. Example: `TO-BULK-20260606-A`, `TO-BULK-20260606-B`.

Auto-generated at session start; user may override the full string with a custom label.

Enable WAL mode on init: `PRAGMA journal_mode=WAL` to prevent read/write contention at 2s insert rate.

---

## Safety Architecture

All safety logic in this section runs entirely on the ESP32 — no network, server, or MQTT dependency.

### Valve and Motor Model

The stepper motor controls a propane valve through a tight coupler. Key physical facts:
- Small turns produce large flame changes — the valve is highly sensitive
- No closed-position sensor exists; position is tracked only via step counting
- When the stepper driver is **enabled** (ENABLE_PIN LOW), coils are energized and the shaft is locked (holding torque). When **disabled** (ENABLE_PIN HIGH), the shaft spins freely for manual adjustment.
- The valve does not drift when the motor is disabled — it holds position passively.
- The propane tank valve remains manually open during operation; the stepper valve is the primary flame control.

### Home Position (Step Zero)

`netSteps = 0` is defined as **minimum stable flame** — the smallest flame the burner reliably holds without going out. A physical mark (paint dot or tape) on the coupler indicates this position. All step counting is relative to this reference.

- **PID operating range:** 0 to `+MAX_OPEN_STEPS` (estimated ~200 based on observed operation; confirmed during commissioning)
- **Emergency close range:** 0 to `-MAX_CLOSE_STEPS` (calibrated during commissioning — see below)
- PID output is hard-clamped to [0, MAX_OPEN_STEPS] before any step pulse is issued. The motor physically cannot exceed these limits regardless of PID output.

### Step Limit Calibration (Commissioning Checklist Item)

Two limits must be calibrated once during initial setup and stored as firmware constants:

**`MAX_CLOSE_STEPS`** — how far CCW past home is safe:
1. Lock motor at home (netSteps = 0)
2. Drive CCW slowly, counting steps, until flame extinguishes
3. Subtract 15% safety margin → `MAX_CLOSE_STEPS`

**`MAX_OPEN_STEPS`** — how far CW from home is safe:
1. Lock motor at home (netSteps = 0)
2. Drive CW slowly, watching flame; stop when flame is clearly excessive (visually unsafe)
3. Subtract 15% safety margin → `MAX_OPEN_STEPS`

Estimated range from field observation: ~200 steps total. Both limits enforced in firmware before any step pulse is issued.

### LittleFS Persistent State

LittleFS stores data in the ESP32's flash memory — the same physical chip that holds the firmware. It survives power loss, power blips, and reboots. Both `netSteps` and `setTemp` are written to LittleFS on every change and restored on boot.

Default `setTemp` if no stored value exists: **450°F** (hardcoded constant matching typical roast target).

### Startup / Ignition Procedure

A new session requires a live server connection before PID starts. Once running, PID sustains independently if the connection drops mid-roast (fire safety). The motor is **disabled on every boot** — this replaces physically unplugging the motor.

```
Boot → ENABLE_PIN HIGH (motor disabled, shaft free)
     → Restore setTemp from LittleFS (default 450°F)
     → OLED: "IGNITION MODE / FREE VALVE"
     → Dashboard: "Ignition Mode — valve free to turn"

User manually opens valve → ignites burner → dials back to reference mark

User presses "Lock & Begin Session" on dashboard (requires server connection)
     → ENABLE_PIN LOW (motor locks at current position)
     → netSteps = 0 saved to LittleFS
     → PID starts
```

**Rain warning:** At "Lock & Begin Session", the server checks the Open-Meteo hourly forecast. If precipitation probability exceeds 5% in any of the next 3 hours, a dismissible warning banner is shown before confirming session start. Roasting in rain is inadvisable; this is a reminder, not a block.

**Future consideration:** A physical button on the device for "Lock & Begin Session" — eliminates the need to have the dashboard open for startup. Not in initial scope.

### Restart Mid-Roast

LittleFS stores both `netSteps` and `setTemp`, so both survive power loss.

- Motor starts disabled (valve holds position — no drift confirmed)
- `netSteps` and `setTemp` restored from LittleFS
- OLED: "RESTART / LOCK TO RESUME"
- Dashboard: "Restart detected — last position: +X steps. Resume or Re-home?"
- **Default action: Resume** — motor locks at stored position, PID continues at stored setTemp
- Manual option: Re-home — user returns valve to reference mark, motor locks at netSteps = 0

### Normal Session End

On "End Roast" from dashboard:
1. PID suspended
2. Drive to netSteps = 0 (minimum stable flame) at normal speed — buys time to close tank valve
3. Motor stays **enabled** at step 0 (holds valve at minimum flame position)
4. Dashboard enters "Post-Roast" state: shows timer counting up from "End Roast" clicked, current temp, and "Close tank valve when ready"
5. Minimum flame will hold temperature around ~250°F — the roaster is still lit until tank valve is manually closed
6. When user closes tank valve, temperature will begin dropping naturally
7. OLED: "BEGIN COOL DOWN" + elapsed time since end
8. System continues logging telemetry through cooldown

### Cooldown Tracking

After session end, the device remains on and continues reading temperature. The server logs telemetry to the session record until temp falls below **200°F**.

- Below 200°F: queue `Roast Notification` alert (alert_type: `cooldown_ready`, message: "Batch is below 200°F — safe to handle")
- OLED: "SAFE TO EJECT" (or for Smoke Infusion: "SAFE TO EJECT" at this point per procedure)
- Dashboard shows cooldown curve in real time

### Temperature Thresholds

| Level | Threshold | Action |
|---|---|---|
| Normal telemetry | Active session, any temp | `progress_update` alert every **10 minutes** |
| Concern zone | >500°F | `temp_alert` every **30 seconds**, PID continues |
| Emergency close | 600°F | Emergency close sequence (see below) |

All thresholds are firmware constants, adjustable without logic changes.

### Emergency Close Sequence

Triggered automatically at 600°F. No network required.

```
1. Drive CCW at maximum safe speed until netSteps = -MAX_CLOSE_STEPS
   (past home, as far toward closed as calibrated limit allows)
2. ENABLE_PIN HIGH — motor releases, valve free for manual intervention
3. PID suspended, session flagged EMERGENCY
4. Queue alert to LittleFS: alert_type "emergency_close", timestamp, last temp
5. OLED: "EMERGENCY / CLOSE TANK VALVE"
6. Dashboard: red emergency state on reconnect
7. Recovery requires explicit manual reset — system does NOT auto-resume
```

**If temperature is NOT dropping during emergency close:** This indicates a stepper, coupler, or valve failure. The firmware detects this by comparing temp readings every 10 seconds during the close sequence. If temp has not dropped by at least 5°F after 30 seconds of driving CCW, it queues a secondary alert (alert_type: `emergency_valve_failure`) and begins sending `temp_alert` notifications every 30 seconds until manually reset. This is an aggressive alert state — the assumption is something is mechanically wrong and immediate human intervention is required.

The $12 stepper motor is considered acceptable collateral damage in an emergency. Burning down the building is not.

### Rate-of-Rise Alarm

A simplified ROR check runs on the device (independent of the server's 60-second rolling ROR):
- Compares current temp to the reading from 30 seconds ago
- If delta > 50°F in 30 seconds while already above 500°F → queues `temp_alert`
- Does not trigger emergency close on its own — that's the 600°F threshold's job

### Blowout Detection

Real-world conditions (wind, cold) cause significant temperature variability — normal noise is ±1–2°F per reading, with gusts causing ±5–10°F swings. A blowout (flame extinguished by wind) produces a sustained, rapid drop distinct from normal noise.

- Server monitors rolling 30-second temperature delta during active sessions
- If temp drops ≥ 30°F in any 30-second window while `netSteps > 0` and session is active → queue `Roast Notification` (alert_type: `blowout_suspected`)
- PID continues running (it will attempt to open valve further, which is correct if the flame relights)
- Alert message: "Possible blowout — temperature dropped 30°F+ rapidly. Check burner."
- Not an emergency close — user investigates and decides. If temp continues dropping, the normal cooldown tracking handles it.

### Alert Queue (Network-Resilient)

All alerts are written to a LittleFS circular buffer (max 20 events) before attempting MQTT publish. On MQTT reconnect, the queue is replayed in order. The server deduplicates by `timestamp + alert_type`. A 10-minute internet outage followed by a reconnect at 550°F sends the alert immediately on reconnect.

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
- `safetyLoop()` — runs every 2s alongside PID; checks temperature thresholds, enforces step limits, triggers emergency close, manages alert queue
- `emergencyClose()` — drives to -MAX_CLOSE_STEPS at max speed, disables motor, writes LittleFS alert; monitors temp drop and escalates to `emergency_valve_failure` alerts if temp not falling
- `loadState()` / `saveState()` — reads/writes `netSteps` AND `setTemp` to LittleFS on every change; both survive power loss

**MQTT command additions:**
```json
{ "cmd": "setMotorEnable", "enabled": false }   — disable motor (free valve for manual)
{ "cmd": "setMotorEnable", "enabled": true }    — enable motor (lock valve)
```
The Manual/Auto dashboard toggle sends `setMotorEnable` before or after `setMode`.

**OLED display — size constraints:**
The OLED is very small (128×64 pixels). Current layout: temperature in large font, a few secondary values in minimum readable font. Nothing can be made smaller than it already is. The OLED must communicate state in 1–3 short lines maximum. All procedural guidance uses brief uppercase labels.

OLED state machine:
| State | Line 1 | Line 2 | Line 3 |
|---|---|---|---|
| Ignition mode | `IGNITION MODE` | `FREE VALVE` | — |
| Running, MQTT OK | `427°F → 450°F` | `AUTO  OK` | `+18 steps` |
| Running, MQTT down | `427°F → 450°F` | `AUTO  NET` | `+18 steps` |
| Restart detected | `RESTART DET.` | `LOCK TO RESUME` | — |
| Cool down | `BEGIN COOL DOWN` | `312°F` | — |
| Safe to eject | `SAFE TO EJECT` | `188°F` | — |
| Emergency | `!! EMERGENCY !!` | `CLOSE TANK` | `600°F` |

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
- `app`: FastAPI + uvicorn, port configured via `APP_PORT` env var. Volumes: `./app:/app` (live reload during dev) + named `roaster-db:/data` for SQLite persistence. Env vars: `MQTT_BROKER`, `DB_PATH`, `KLAVIYO_API_KEY`, `KLAVIYO_PROFILE_EMAIL`, `WEATHER_ZIP`, `OPEN_METEO_LAT`, `OPEN_METEO_LON`, `APP_PORT`.

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

### API-First Design

All dashboard functionality is backed by the REST API. The UI is a thin client that calls the same endpoints any future agent or external tool would use. No business logic lives in HTML/JS — only presentation.

### Weather Integration (`weather.py`)

Fetches current conditions from **Open-Meteo** (free, no API key) at session start. Zip code 75248 is the default; configurable via `WEATHER_ZIP` env var. Lat/lon resolved once from zip and cached.

```python
# Open-Meteo endpoint (no auth required):
GET https://api.open-meteo.com/v1/forecast
    ?latitude={lat}&longitude={lon}
    &current=temperature_2m,wind_speed_10m
    &temperature_unit=fahrenheit
    &wind_speed_unit=mph
```

Weather is fetched once at "Lock & Begin Session" and stored in the session record. Not polled during the roast.

### REST Endpoints
```
# Sessions
POST /api/sessions/start              — profile, lot_id (optional override), notes; fetches weather
POST /api/sessions/{id}/end           — color_rating, color_notes, operator_notes
GET  /api/sessions                    — list (50 most recent, includes lot_id + weather)
GET  /api/sessions/{id}               — full session detail
GET  /api/sessions/{id}/telemetry     — time-series for historical chart
POST /api/sessions/{id}/photo         — upload batch color JPEG (multipart)

# Profiles
GET/POST         /api/profiles        — list / create
GET/PUT/DELETE   /api/profiles/{id}   — detail / update / delete

# Commands (validated → MQTT publish)
POST /api/commands                    — CommandPayload → roaster/commands topic

# Status
GET  /api/status                      — latest telemetry snapshot + session state

# Events
POST /api/events                      — manually log a roast event

# Color reference library
GET  /api/color-refs                  — list all reference photos by wood type
POST /api/color-refs                  — upload a reference JPEG with label + wood type
DELETE /api/color-refs/{id}           — remove a reference photo

# Weather (on-demand fetch, not stored)
GET  /api/weather                     — current conditions for configured zip
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
| `alert_type` | When | Frequency |
|---|---|---|
| `progress_update` | Active session, temp ≤ 500°F (normal operation) | Every **10 minutes** |
| `temp_alert` | Temp > 500°F (concern zone) | Every **30 seconds** |
| `emergency_close` | Emergency close sequence triggered (≥ 600°F) | Once on trigger |
| `emergency_valve_failure` | Temp not dropping during emergency close | Every **30 seconds** until manual reset |
| `profile_step` | Profile-scheduled offset reached (cut flame, eject, lid on, etc.) | On schedule |
| `cooldown_ready` | Temp drops below 200°F post-session | Once |
| `blowout_suspected` | Temp drops ≥30°F in 30s during active session | Once per event |

**Properties always included:** `session_id`, `wood_type`, `current_temp`, `elapsed_min`, `alert_type`, `message`, `subject`.

`subject` is included in the event payload so Klaviyo's conditional blocks can set the email subject line without needing separate flows. Email body content (including future HTML formatting) is controlled by Klaviyo template conditional blocks keyed on `alert_type`.

---

## Maintenance Tracking

A running roast count is stored in the database. After every 5 completed sessions, a **toast banner** appears on the dashboard at next login prompting the user to perform machine maintenance before starting a new roast. The banner is dismissible (user can acknowledge and proceed).

**Maintenance checklist (shown in the banner):**
- Compressed air on all burners (clear debris and dust)
- Machine greasing
- Machine oiling
- Chain oiling
- General cleaning

Maintenance acknowledgement is logged as a `roast_events` entry (event_type: `maintenance_logged`) so the interval resets. The 5-roast threshold is configurable.

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
| Toasted Cherry | 450°F | 45 min | 35m: "10 min until flame cut", 45m: "Cut flame", 75m: "Check cooling" — identical to Toasted Oak; stored separately so procedures can diverge independently |

---

## Frontend Features (`static/index.html` + `static/app.js`)

**Design:** Mobile-first. All controls and live data are designed for phone/tablet use. Historical analysis views (charts, overlays, color consistency table) may use wider layouts on desktop — this is the one area where desktop-first is acceptable.

**Live Dashboard:**
- Large current temp display + ROR (°F/min)
- Set temp with +5/-5 buttons
- Manual/Auto toggle — switching to Manual **disables the motor** (ENABLE_PIN HIGH), freeing the valve for hand adjustment. CW/CCW buttons and step size slider are hidden in Manual mode (motor is decoupled — buttons have no effect). Switching back to Auto re-enables the motor.
- Real/Sim toggle — **disabled during an active session with live telemetry**; can only switch to sim when no real device telemetry is being received
- Elapsed time + net steps
- Dual-axis Chart.js: temp (left) + ROR (right, dashed); chart animation is a user-toggleable setting (default off for performance during live updates)
- WebSocket with 3s auto-reconnect
- Post-roast state: timer counting up since "End Roast", current temp, "Close tank valve when ready" prompt
- Cooldown tracking view: temp curve continues until <200°F, then `cooldown_ready` alert fires

**Session Management:**
- "Lock & Begin Session" button (requires server connection) → modal:
  - Select profile
  - Lot ID (auto-assigned, optional override with custom label)
  - Batch volume (default: 5 gal — standard batch size; editable)
  - Ambient conditions auto-fetched from Open-Meteo (shown, not editable)
  - Operator notes
- "End Roast" button → modal: color rating (with color reference swatches), color notes, batch photo upload (JPEG), operator notes
- Active session shown in header with elapsed time and lot ID

**History Tab** (desktop-friendly layout acceptable):
- Table of past sessions: lot ID, date, profile, peak temp, color rating, weather (temp + wind), status
- Click row → loads historical chart (x-axis = elapsed minutes, not wall clock)
- Overlay toggle: select a past roast to overlay on any chart for visual comparison

**Color Consistency Table** (desktop-friendly layout acceptable):
- Filters by wood type / profile
- Shows: lot ID, date, peak temp, color rating, batch photo thumbnail, weather, notes
- Color reference photo library: upload JPEG swatches per wood type with color label (Light/Medium/Dark/Extra Dark) for side-by-side comparison

**PID Tuning Panel** (`<details>` collapsed by default):
- Editable Kp, Ki, Kd inputs → sends `setPid` command to ESP32
- Each value has a plain-language tooltip/description explaining what it controls in non-technical terms (e.g. Kp: "How aggressively the system responds to being off-target. Higher = faster response but more overshoot.")

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

1. Flash firmware → OLED shows `IGNITION MODE / FREE VALVE` → motor shaft spins freely by hand
2. Send "Lock & Begin Session" command via dashboard → OLED switches to temp display → stepper locks
3. `mosquitto_sub -t roaster/telemetry` on broker → confirms JSON telemetry every 2s
4. `docker compose up` → `http://PRODUCTION_HOST:8060` loads dashboard → live temp chart updates
5. Press CW/CCW on dashboard (manual mode) → ESP32 blue LED blinks → stepper moves
6. Start a session with "Toasted Oak" profile → confirm weather auto-fetched → lot ID auto-assigned
7. Run simulator anomaly: `python simulator.py --scenario runaway` → confirm 500°F alerts fire every 30s, 600°F triggers emergency close
8. Emergency close: confirm OLED shows `!! EMERGENCY !!`, motor disables, alert queued to LittleFS
9. Kill MQTT mid-session → confirm PID keeps running → reconnect → confirm queued alerts replay
10. End session → upload batch photo → verify in history tab with lot ID and weather data
11. Check SQLite: `sqlite3 /data/roaster.db "SELECT lot_id, ambient_temp_f, wind_speed_mph FROM roast_sessions"`
12. Load historical session → overlay a second session → confirm dual curves on chart
