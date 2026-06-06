# Build Progress — ESP32 Roaster Controller

This doc tracks milestone status and checkpoint notes for resuming across sessions.
Architecture decisions and specs live in `docs/architecture.md`.

---

## Pre-Build — COMPLETE

- [x] Architecture designed and documented (`docs/architecture.md`)
- [x] Git repo initialized, pushed to GitHub (`BourbonBaggers/esp32-roaster-controller`)
- [x] Directory skeleton created: `docker/`, `firmware/`, `docs/`
- [x] `.gitignore` configured (env files, db, PlatformIO build output, macOS)
- [x] `.claude/settings.json` with all bash/docker/git permissions (no prompts during build)
- [x] Security incident resolved: server IP/user removed from git history via `git-filter-repo`

---

## Phase 1: Mac-Local Full Simulation

Goal: `docker compose --profile sim up` on Mac → everything runs locally, browser at `localhost`.
No physical device. No VM. Fast iteration.

---

### Milestone 1: Infrastructure Layer
**Status: NOT STARTED**

Files to create:
- `docker/docker-compose.yml`
- `docker/mosquitto/mosquitto.conf`
- `docker/.env.example`

Done when: `docker compose up` (without sim profile) starts broker + app containers without errors.

---

### Milestone 2: Simulator
**Status: NOT STARTED**

Files to create:
- `docker/simulator/simulator.py`
- `docker/simulator/Dockerfile`
- `docker/simulator/requirements.txt`

Key behaviors:
- Thermal inertia physics (realistic 15–30 min ramp, not instant)
- Temperature noise ±5–10°F, more with simulated wind
- Blowout scenario: 30°F drop in 30s on trigger
- Responds to all MQTT commands (`setTemp`, `setMode`, `moveStepper`, `setSimTemp`, etc.)
- `--speed N` flag for fast-forward testing (10× = 45-min roast in 4.5 min)
- Simulated weather params (wind speed, ambient temp) injected into telemetry

Done when: `docker compose --profile sim up` → `mosquitto_sub -t 'roaster/telemetry'` shows
realistic JSON every 2s with temp, setTemp, netSteps, mode, alertFlags.

---

### Milestone 3: Database Layer
**Status: NOT STARTED**

Files to create:
- `docker/app/database.py`

Key behaviors:
- SQLite with WAL mode (`PRAGMA journal_mode=WAL`)
- Schema: `roast_sessions`, `roast_profiles`, `color_reference_photos` (see architecture.md)
- Pre-seed 5 default profiles (see architecture.md profile list)
- Telemetry pruning: rows older than `TELEMETRY_RETAIN_DAYS` deleted on startup
- `aiosqlite` for async access

Done when: DB file creates cleanly on container start, profiles present, WAL mode confirmed.

---

### Milestone 4: MQTT Handler
**Status: NOT STARTED**

Files to create:
- `docker/app/mqtt_handler.py`

Key behaviors:
- `paho-mqtt 2.0.0` client, thread bridge via `asyncio.run_coroutine_threadsafe`
- Subscribe `roaster/telemetry` (QoS 0) → persist to DB + broadcast via WebSocket
- Publish to `roaster/commands` (QoS 1) on API request
- Server-side deduplication of LittleFS alert replays (timestamp + alert_type)
- Reconnect loop with exponential backoff

Done when: telemetry messages persist to DB rows, commands relay to broker on API call.

---

### Milestone 5: FastAPI Core
**Status: NOT STARTED**

Files to create:
- `docker/app/models.py`
- `docker/app/main.py`

Endpoints (all backed by REST, dashboard calls these — not direct DB):
- `GET /api/telemetry/latest` — last known device state
- `GET /api/telemetry/history?since=` — historical readings for charts
- `POST /api/command` — relay command to device via MQTT
- `GET /api/session` — active session details
- `POST /api/session/start` — start roast (lot ID, profile, volume, tea bags)
- `POST /api/session/end` — end roast (color rating, notes)
- `GET /api/profiles` — list roast profiles
- `POST /api/profiles` — create custom profile
- `GET /api/weather` — current conditions + rain warning
- `WebSocket /ws` — push telemetry + alerts to dashboard in real time

Done when: all endpoints return correct Pydantic shapes, WebSocket emits on telemetry arrival.

---

### Milestone 6: Weather Integration
**Status: NOT STARTED**

Files to create:
- `docker/app/weather.py`

Key behaviors:
- Open-Meteo free API, no key required
- Lat/lon from `.env` (zip 75248 = approx 32.9°N, 96.8°W)
- Cache response 10 minutes (don't hammer free API)
- Rain warning: flag if precipitation probability ≥ 5% within next 3 hours
- Returns: temp_f, wind_speed_mph, condition, rain_warning bool + hours_until_rain

Done when: `GET /api/weather` returns correct data for Dallas, rain warning logic fires correctly.

---

### Milestone 7: Klaviyo Handler
**Status: NOT STARTED**

Files to create:
- `docker/app/klaviyo_handler.py`

Key behaviors:
- Single event name: `Roast Notification`
- `alert_type` property drives conditional content blocks in Klaviyo template:
  - `session_start` — roast started, lot ID, profile, weather
  - `temp_concern` — above 500°F (every 5 min while above)
  - `temp_emergency` — 600°F hit, close command sent
  - `blowout_detected` — 30°F drop in 30s
  - `session_end` — completed, peak temp, color rating
  - `rain_warning` — precip forecast during roast window
- Additional properties: `temp`, `set_temp`, `lot_id`, `profile_name`, `peak_temp`,
  `ambient_temp`, `wind_speed`, `color_rating`, `operator_notes`
- API key from `KLAVIYO_API_KEY` env var
- Profile email from `KLAVIYO_PROFILE_EMAIL` env var

Done when: event posts to Klaviyo test environment with correct payload for each alert_type.

---

### Milestone 8: Profile Scheduler
**Status: NOT STARTED**

Files to create:
- `docker/app/profile_scheduler.py`

Key behaviors:
- Background asyncio task, runs while session is active
- Below 500°F: status email every 10 minutes
- Above 500°F (concern zone): alert every 30 seconds
- 600°F trigger: send `setMode auto=false` + close stepper command immediately, fire
  `temp_emergency` Klaviyo event, no further auto-close after one attempt
- Blowout (30°F drop in 30s): send blowout alert, do not auto-close (operator decides)
- Rain warning check: every 30 minutes during active session

Done when: correct cadence observed in logs with simulated temperature progression.

---

### Milestone 9: Dashboard UI
**Status: NOT STARTED**

Files to create:
- `docker/app/static/index.html`
- `docker/app/static/app.js`

Key behaviors:
- Mobile-first (375px baseline), hamburger slide-out drawer navigation
- Sections: Live Dashboard, Session, Manual Control, Profiles, History, Settings
- Live dashboard: current temp (large), set temp, net steps, mode (auto/manual), alert banner
- Temperature chart: last 60 minutes, animation toggle
- Session: lot ID entry (auto-generate or custom), profile picker, volume, tea bags count
- Manual control: CW/CCW step buttons, step size (1/5/10/25), motor lock toggle
- History: roast list with color rating photos, peak temps, lot IDs
- WebSocket for real-time updates (no polling)
- API-first: every action calls a REST endpoint

Done when: live temp updates on dashboard, manual step buttons send commands,
session start/end flows complete, tested on iPhone Safari viewport.

---

### Milestone 10: App Container + Full Compose
**Status: NOT STARTED**

Files to create:
- `docker/app/Dockerfile`
- `docker/app/requirements.txt`

Done when: `docker compose --profile sim up --build` starts all services clean,
no import errors, no volume permission issues, logs show telemetry flowing.

---

## Phase 1 Verification Checkpoint
**Status: NOT STARTED**

Acceptance criteria:
- [ ] `docker compose --profile sim up --build` — all three services healthy
- [ ] Simulator runs 45-min roast at `--speed 10` (4.5 min wall clock)
- [ ] Temp curve realistic: slow ramp, holds, natural variance
- [ ] Blowout scenario triggers correct alert in logs
- [ ] 600°F scenario sends close command + Klaviyo event
- [ ] Alert cadence correct (10-min below 500°F, 30-sec above)
- [ ] Dashboard loads on iPhone viewport, hamburger nav works
- [ ] Session start/end completes, lot ID appears in history
- [ ] `GET /api/telemetry/history` returns complete roast data
- [ ] Color reference photo upload works

---

## Phase 2: Dress Rehearsal (Mac Simulator → VM Server)
**Status: NOT STARTED**
**Prerequisite:** Phase 1 verification complete

Steps:
1. Copy docker stack to VM (broker + app, no `--profile sim`)
2. Create `docker/.env` on VM with real credentials (Klaviyo key, lat/lon, etc.)
3. Start stack on VM: `docker compose up -d`
4. Run simulator on Mac: `python3 simulator.py --broker PRODUCTION_HOST`
5. Browse dashboard from iPhone on local network
6. Send test Klaviyo events, verify receipt

Done when: simulator on Mac talks to VM, dashboard loads on phone, Klaviyo events arrive.

---

## Phase 3: Hardware (DEFERRED)
**Status: DEFERRED — no physical device in build environment**
**Prerequisite:** Phase 2 complete

Firmware changes needed (all documented in `architecture.md` firmware section):
- ArduinoJson v7 (JsonDocument, not StaticJsonDocument)
- LittleFS for netSteps + setTemp persistence
- ENABLE_PIN control (HIGH=free on boot, LOW=locked on session start)
- mDNS resolution (`roaster-hub.local` via ESPmDNS)
- Circular buffer alert replay on MQTT reconnect (20-event max, deduplicated server-side)
- OLED state machine with uppercase short labels
- `setSimMode` command support (sim temp injection path)
- Emergency close sequence (drive to 0 netSteps, then ENABLE_PIN HIGH)
- Blowout detection (30°F drop in 30s)

---

## Key Config Values (fill in `.env` locally — never commit)

```
APP_PORT=8060
MQTT_BROKER=mosquitto          # service name inside Docker; use VM IP for Phase 2
DB_PATH=/data/roaster.db
KLAVIYO_API_KEY=               # from Klaviyo account
KLAVIYO_PROFILE_EMAIL=         # email address to receive alerts
WEATHER_ZIP=75248
OPEN_METEO_LAT=32.9
OPEN_METEO_LON=-96.8
TELEMETRY_RETAIN_DAYS=365
```
