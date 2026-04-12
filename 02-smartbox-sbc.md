# 02 -- SmartBox / SBC Edge System

> Edge computing platform for local inverter monitoring and control via Raspberry Pi and M5Stack CoreMP135 devices.
> Last updated: 2026-04-12

---

## Overview

SmartBox (SB) is an edge computing device (Raspberry Pi) installed at a customer's PV site. It communicates directly with inverters via Modbus TCP, stores data locally in SQLite, and syncs with PgePilot Cloud via JSON-RPC over a reverse SSH tunnel.

| Component | Description |
|-----------|-------------|
| **SmartBox (SB)** | Physical device at customer site (RPi 4 or M5Stack CoreMP135) |
| **SBC (SmartBox Controller)** | Software running on the SB |
| **VPS** | ra-energity.cz -- management server, SSH tunnel endpoint |
| **sb-manager** | Web UI for managing all SmartBox devices |

---

## VPS: ra-energity.cz

| Property | Value |
|----------|-------|
| IP | 195.201.19.103 |
| OS | Ubuntu 24.04.3 LTS (x86_64) |
| RAM | 3.7 GB |
| Disk | 38 GB SSD (28 GB free) |
| SSH | `ssh limited@ra-energity.cz` |
| Node.js | v18.19.1 |
| SSL | Let's Encrypt wildcard *.ra-energity.cz (expires 2026-06-06) |

### Services on VPS

| Service | Type | Port | Path | Purpose |
|---------|------|------|------|---------|
| sb-manager | PM2 | 3055 | /opt/sb-manager | Web management of all SB devices |
| pgepilot-smartbox-sim | PM2 | 5001 | /opt/sbcsim | SmartBox simulator for development |
| nginx | systemd | 80, 443 | -- | Reverse proxy, SSL termination |
| sb-router | systemd | 9100 (localhost) | /opt/sb-router | Subdomain routing to SSH tunnels |

### Runtime Snapshot (verified 2026-04-12)

- `sb-manager`: clean `main@ef3261b`, PM2 online, live cloud identity sync for SmartBox provisioning is working
- PM2 restart counter is `168` as of `2026-04-12 22:20 CEST`; current uptime is short because of same-day deploy/restart
- `sb-router`: clean `main@4c4a7ad`, systemd active
- `pgepilot-smartbox-sim`: PM2 online on port 5001

### DNS Routing

```
Internet -> ra-energity.cz:443 -> nginx
  +-- sb-manazer.ra-energity.cz  -> 127.0.0.1:3055 (sb-manager)
  +-- sb1.ra-energity.cz         -> sb-router:9100 -> tunnel :20000 -> SB1:80
  +-- doma2.ra-energity.cz       -> sb-router:9100 (alias) -> :20000
  +-- sb1-ops.ra-energity.cz     -> sb-router:9100 -> tunnel :20005 -> SB1:3002
  +-- sb1-health.ra-energity.cz  -> sb-router:9100 -> tunnel :20009 -> SB1:3007
```

Config files: `/etc/sb-router/config.json`, `/etc/nginx/conf.d/01-sb-router-host-map.conf`

### VPS Users

| User | Purpose |
|------|---------|
| limited | Main working account, PM2, git push |
| ra_tunnel | Receives reverse SSH tunnels from SB devices |
| ops | Operations account |
| root | Root access |

### SSH Tunnel Keepalive (added 2026-04-12)

VPS sshd is configured to detect dead reverse tunnels automatically:

```
# /etc/ssh/sshd_config (appended)
ClientAliveInterval 30    # ping client every 30s
ClientAliveCountMax 3     # disconnect after 3 missed (= 90s)
```

Without this, rebooted SB devices leave zombie tunnel sessions on VPS, blocking port re-binding for up to 2h (TCP keepalive default). With this setting, stale tunnels are cleaned up within 90s.

**Manual cleanup** (if needed): `sudo pkill -u ra_tunnel` on VPS kills all tunnel sessions. All SBs reconnect automatically within 30-90s (systemd RestartSec + backoff).

Backup: `/etc/ssh/sshd_config.bak-20260412`

### Git Push from VPS

```bash
GIT_SSH_COMMAND="ssh -i /home/limited/.ssh/githolbytlo -o IdentitiesOnly=yes" git push origin <branch>
```

---

## Hardware Watchdog (all SB devices)

All SB devices have hardware watchdog enabled (added 2026-04-12):

```ini
# /etc/systemd/system.conf
RuntimeWatchdogSec=30
```

After `systemctl daemon-reexec`, systemd kicks the hardware watchdog every ~15s. If systemd freezes for 30s (kernel panic, OOM, IO deadlock), the watchdog chip hard-resets the device.

| Device | Watchdog chip | Timeout |
|--------|--------------|---------|
| Raspberry Pi 4 (sb1) | Broadcom BCM2835 | 1 min (HW min) |
| M5Stack CoreMP135 (sb4, sb7) | STM32 IWDG | 30s |

**This must be enabled during provisioning of every new SB device** -- see provisioning runbook.

Backup: `/etc/systemd/system.conf.bak-20260412` on each device.

---

## SmartBox 1 (SB1) -- Raspberry Pi 4

| Property | Value |
|----------|-------|
| Hardware | Raspberry Pi 4 Model B Rev 1.5 |
| OS | Debian 13 (trixie), aarch64 |
| RAM | 1.8 GB (~123 MB free, ~815 MB available with cache) |
| Swap | 1.8 GB (251 MB used, verified 2026-04-04) |
| Disk | 59 GB SD card (48 GB free) |
| WiFi IP | 192.168.0.51 |
| Python | 3.13.5 (venv) |
| Node.js | v20.19.2 |
| Inverter | GoodWe ET+ at 192.168.0.70:502 (Modbus TCP) |
| Runtime content | `/opt/energity/sb`, rsync-deployed runtime. Audited key file hashes currently match local `sb/devva@45a2fc0` (`smartbox_service.py`, `rpc_client/models.py`, `install_systemd_services.sh`, `base_device.py`). |
| Watchdog | BCM2835, RuntimeWatchdogSec=30 (enabled 2026-04-12) |

### Access

```bash
# From VPS:
ssh -p 20002 root@127.0.0.1

# Direct from local PC (SSH jump):
ssh -J limited@ra-energity.cz -p 20002 root@127.0.0.1
```

### Reverse SSH Tunnel (SB1 -> VPS)

```
SB1 (192.168.0.51)                 VPS (195.201.19.103)
  SSH tunnel ------>               ra_tunnel@VPS:22

  VPS :20000  <->  SB1 :80     nginx (web UI)
  VPS :20002  <->  SB1 :22     SSH
  VPS :20003  <->  SB1 :3000   config-api
  VPS :20004  <->  SB1 :3001   rpc-client
  VPS :20005  <->  SB1 :3002   sb-ops-agent
  VPS :20009  <->  SB1 :3007   health aggregator
```

**Port schema for multiple SBs**: `base = 20000 + (N-1) * 10`, offsets: +0=nginx, +2=ssh, +3=config, +4=api, +5=ops, +9=health

Verified active SSH ports on VPS in this session:
- SB1: `20002`
- SB4: `20032`
- SB7: `20062`

> **Note (2026-04-04)**: Ports 20050-20055 are active on VPS -- this is a test SmartBox at a customer site. Not yet production, will be resolved gradually. Ask about current status before working on it.

### Services on SB1 (all systemd, root)

| Service | Port | Bind | Tech | Purpose |
|---------|------|------|------|---------|
| energity-device-controller | 5010 | 127.0.0.1 | Python/Flask | Modbus polling, REST API for devices |
| energity-local-db | 5011 | 127.0.0.1 | Python/Flask | SQLite sensor data storage |
| energity-logs-db | 5012 | 127.0.0.1 | Python/Flask | SQLite log storage |
| energity-config-api | 3000 | 127.0.0.1 | Python/Flask | Device configuration |
| energity-web-interface | 5000 | 127.0.0.1 | Python/Flask | User/service web UI |
| energity-health | 3007 | **0.0.0.0** | Python | Health check aggregator |
| energity-rpc-client | 3001 | **0.0.0.0** | Python | Cloud communication (JSON-RPC) |
| sb-ops-agent | 3002 | 127.0.0.1 | Node.js | Proxy for sb-manager |
| energity-comm-controller | -- | -- | Python | Orchestrates data flows and commands |
| ra-tunnel | -- | -- | SSH | Reverse SSH tunnel to VPS |
| nginx | 80, 8080 | 0.0.0.0 | nginx | Proxy to Flask web interface |

**Warning**: `energity-health` and `energity-rpc-client` bind to 0.0.0.0 (should be 127.0.0.1).

Observed on 2026-04-09:

- `sb-ops-agent` is active and running from `/opt/energity/sb-ops-agent/server.mjs`
- `sb-test-3000` and `sb-test-3001` are still in auto-restart loops

### Useful Commands

```bash
systemctl status energity-*          # Status of all services
systemctl restart energity-device-controller
journalctl -u energity-device-controller -f
```

---

## SmartBox 4 (sb4) -- M5Stack CoreMP135

| Property | Value |
|----------|-------|
| Hardware | M5Stack CoreMP135 (STM32MP135D, ARMv7 Cortex-A7) |
| OS | Debian 12 (bookworm), armv7l |
| RAM | ~512 MB |
| Disk | 15 GB SD card (13 GB free) |
| Ethernet IP | 192.168.0.167 |
| Python | 3.11.2 (venv) |
| Location | Dev box (doma, stejna LAN jako sb1) |
| Inverter | GoodWe at 192.168.0.70:502 (sdileny se sb1, enabled: false -- viz poznamka) |
| Runtime content | `/opt/energity/sb`, rsync-deployed runtime. Audited key file hashes currently match local `sb/devva@45a2fc0`. |
| Watchdog | STM32 IWDG, RuntimeWatchdogSec=30 (enabled 2026-04-12) |

### Access

```bash
ssh -J limited@ra-energity.cz -p 20032 root@127.0.0.1
```

### Reverse SSH Tunnel (sb4 -> VPS)

```
VPS :20030  <->  sb4 :80     nginx (web UI)
VPS :20032  <->  sb4 :22     SSH
VPS :20033  <->  sb4 :3000   config-api
VPS :20034  <->  sb4 :3001   rpc-client
VPS :20035  <->  sb4 :3002   sb-ops-agent
```

### Services on sb4 (all systemd, root)

Same as SB1 layout (7 energity-* services + sb-ops-agent + ra-tunnel + nginx), all `active running` (verified 2026-04-12). sb-ops-agent reports allUp=true, 6/6 services.

### SmartBox auth rollout status (verified 2026-04-12)

- SB1, SB4, and SB7 now authenticate against `https://service.pgepilot.cz`
- SB1, SB4, and SB7 now each have their own SmartBox cloud identity bundle: `login + smartbox_id + collection_point_id + device_id`
- legacy `https://auth.pgepilot.cz` remains active for the remaining boxes
- old shared auth row (`sbx_deye25`) may still exist in `pge_control.cp_connector_auth`, but current boxes no longer use it

### GoodWe Modbus TCP collision warning

sb4 and sb1 share the same LAN and the same GoodWe inverter (192.168.0.70). **Only ONE SmartBox can poll the inverter at a time.** Two simultaneous Modbus TCP clients cause `Connection reset by peer` loops on both sides -- the inverter firmware resets both connections within minutes.

**Rule:** On sb4 `devices_config.yaml`, main inverter is `enabled: false`. Only enable for testing when sb1 DC is stopped first.

### Deye driver (merged 2026-04-11, commit 5322f3f)

`origin/devva` contains DeyeInverterDriver, but live SB1 does not have this commit yet. sb4 was used for the smoke test and SB1 can receive the same additive deploy later. Files added:
- `device_controller/drivers/communication/solarman_driver.py` (Solarman V5 protocol)
- `device_controller/drivers/devices/deye_inverter.py` (DeyeInverterDriver)
- `device_controller/functions/commands_deye.yaml` (13 commands)
- `device_controller/functions/functions_deye_poll.yaml` (4 poll categories)
- `device_controller/drivers/devices/inverter_modbus_registers/modbus_reg_deye.yaml` (36 registers, SUN-xK-SG04LP3-EU)

Deye driver is opt-in: only activates when `devices_config.yaml` specifies `device_driver: DeyeInverterDriver`.

Smoke test passed on sb4 (2026-04-11): imports OK, 36 registers parsed, 13 commands loaded, 4 poll functions loaded, connect failure to TEST-NET-1 IP was graceful (no crash).

---

## LCD Dashboard / Display Support

SmartBox LCD dashboard must use the display hardware that is physically present on the machine. The application must not select the display type by box label such as `sb4` or `sb13`.

### Runtime rule

- one shared dashboard runtime is deployed on the box as `/opt/energity/sb-ops-agent/display_dashboard.py`
- at startup it must autodetect framebuffer driver, framebuffer path, panel resolution, and touch input that actually exist on the device
- provisioning prepares kernel overlay and dependencies for the hardware, but the app itself must not hardcode a per-box display choice
- if no supported display is present, `energity-display.service` should fail cleanly without affecting the core SB services

### Supported display hardware

| Platform | Display | Framebuffer | Touch | Provisioning note |
|----------|---------|-------------|-------|-------------------|
| M5Stack CoreMP135 (`sb4`, `sb7`) | Built-in ILI9342C, 320x240 | `/dev/fb1` | `evdev` swipe, usually `/dev/input/event0` | No extra display config; driver is loaded by the board image |
| Raspberry Pi + 3.5" TFT HAT (`sb13` class) | ILI9486, 480x320 | `/dev/fb0` or `/dev/fb1` | Primary: SPI polling on `spidev0.1`; fallback: `evdev` | Requires `tft35a` overlay and display dependencies during provisioning |

### UI conventions

- 3 pages: `Status`, `Energy Flow`, `Data`
- bottom navigation uses dots plus visible `<` and `>` arrows on both sides as a navigation hint
- alias header comes from `/opt/energity/sb-ops-agent/device_alias.txt`
- dashboard should detect both `fb0` and `fb1` by driver identity, not by fixed device number

---

## Microservice Architecture (4 Core Services)

### 1. Device Controller (Port 5010)

Direct communication with physical devices via Modbus TCP.

**Components**:
- `device_service.py` -- Flask REST API entry point
- `device_manager.py` -- Device lifecycle, poll loop
- `config_loader.py` -- YAML -> DeviceConfig
- `devices_config.yaml` -- Device configuration
- `constraints_manager.py` -- Command limits and constraints
- `functions/` -- Poll function YAML configs
- `drivers/communication/` -- Modbus TCP/RTU drivers (thread-safe TCP, RTU lacks thread safety)
- `drivers/devices/` -- Device drivers: `base_device.py` -> `modbus_device.py` -> `inverter.py` -> `goodwe_inverter.py`

**Supported devices** (driver hierarchy):
```
base_device.py (ABC)
  +-- modbus_device.py (register read/write, scaling)
  |     +-- inverter.py (inverter abstraction)
  |     |     +-- goodwe_inverter.py  [ACTIVE on SB1, sb4]
  |     |     +-- deye_inverter.py    [READY, merged 2026-04-11 for sb7]
  |     |     +-- solax_inverter.py   [READY]
  |     |     +-- victron_inverter.py [READY]
  |     +-- gpio_device.py (GPIO driver)
  |           +-- relay.py (relay driver)
```

**Communication drivers:**
```
base_driver.py (BaseCommunicationDriver)
  +-- modbus_tcp.py (ModbusTCPDriver)     [thread-safe, production]
  +-- solarman_driver.py (SolarmanDriver) [Solarman V5 protocol for Deye LSW-3 dongle]
  +-- modbus_rtu.py                       [lacks thread safety]
```

**Active device configuration** (SB1, verified 2026-04-04):
```yaml
# Device 1: GoodWe inverter (ENABLED)
inverter:
  driver: GoodWeInverterDriver
  protocol: Modbus TCP
  IP: 192.168.0.70:502, unit_id: 247
  polling: 5s
  poll functions:
    - read_telemetry_data (10s)    # PV, load, battery, grid, SoC
    - read_energy_data (60s)       # Daily kWh counters
    - read_grid_detail (60s)       # Voltage and power per phase
    - read_ems_data (60s)          # Work mode, feed power
    - read_battery_health (60s)    # BMS limits, SoH
    - read_temperature_data (60s)  # Air, module, heatsink
    - read_status_data (60s)       # Warnings, errors

# Device 2: GPIO Relay (DISABLED)
relay:
  driver: RelayDriver
  gpio_pin: 17
  active_high: true
  enabled: false

# Device 3: inverter_test_static (legacy test, DISABLED)
```

### 2. Local Database (Port 5011)

SQLite storage for sensor data with 1-day retention.

- `sensor_data_service.py` -- Flask API
- `sensor_data.db` -- SQLite database
- Endpoints: `POST /sensor-data`, `GET /api/sensor-data/latest`, `GET /api/sensor-data/range`

### 3. RPC Client (Port 3001)

Communication with PgePilot Cloud via JSON-RPC 2.0 over HTTPS.

- `main.py` -- FastAPI application
- JWT authentication against a configurable auth endpoint
- Methods: `smartboxSendData`, `smartboxPoll`
- Credentials configured in `rpc_client_config.yaml`

Current production auth split:
- Current/default path for new and migrated SmartBoxes: `https://service.pgepilot.cz`
- Legacy compatibility path for older boxes not yet migrated: `https://auth.pgepilot.cz`
- Both use the same `/login`, `/refresh-token`, and `/verify-token` contract

### 4. Communication Controller (no dedicated port)

Orchestrates data flows between all other services.

- `smartbox_config_loader.py` -- Configuration management
- Poll loop: reads from Device Controller, pushes to Local DB
- Send loop: reads from Local DB, sends via RPC Client to Cloud
- Command loop: polls Cloud for commands, dispatches to Device Controller

---

## Data Flow

```
GoodWe inverter (192.168.0.70:502)
  | Modbus TCP (pymodbus), poll every 5-10s
  v
Device Controller (:5010) -- read registers, apply scaling
  | HTTP POST /sensor-data
  v
Sensor DB (:5011) -- SQLite, 1-day retention
  | HTTP GET /sensor-data/range
  v
Communication Controller -- field mapping, poll/send loops
  | smartboxSendData (every 5s)
  v
RPC Client (:3001) -- FastAPI, JWT auth, JSON-RPC 2.0
  | HTTPS auth to service.pgepilot.cz (current) or auth.pgepilot.cz (legacy compatibility)
  | HTTPS RPC to service.pgepilot.cz/rpc
  v
PgePilot Cloud -- stores in rpc_kv, processes telemetry
```

---

## TOU (Time-of-Use) Control Engine

Three-layer time-based control per Device. Precedence (lowest to highest):

```
defaults -> TOU schedule -> dateTime overrides -> direct runtime command
```

### RPC Methods (sb.* namespace)

| Method | Purpose |
|--------|---------|
| `sb.setDefaults` | Baseline parameters (mode, exportLimitW, gridTargetW, batteryTargetW) |
| `sb.setTouSchedule` | Recurring rules (months, days, hours -> parameters) |
| `sb.setDateTimeOverrides` | Specific date+time override |
| `sb.queueCommand` | Direct command (setMode, setExportLimit, setGridTarget, setBatteryTarget, clearTargets) |
| `sb.getState` | Returns actual / desired / effective state |
| `sb.getTouPlan` | Get current TOU plan |
| `sb.evaluateTou` | Evaluate TOU for a specific time |
| `sb.listCommands` | List pending commands |
| `sb.listEvents` | List device events |

### State Model

- **actual** -- what the physical device is doing
- **desired** -- what the control system wants
- **effective** -- desired after applying HW constraints (limits, ranges)
- **SELF_USE** -- when no target is active, battery automatically covers load from PV surplus

### Status

- **v0.1** = PLANT device (inverter control) -- **in progress**
- **v0.2** = + BOILER, EVSE, POOL, HVAC (future)

---

## Web Access (SB1)

| URL | Login | Role |
|-----|-------|------|
| https://sb1.ra-energity.cz | [REDACTED] | user -- dashboard |
| https://sb1.ra-energity.cz | [REDACTED] | technician -- service panel |
| https://sb1.ra-energity.cz | [REDACTED] | admin -- full access |
| https://doma2.ra-energity.cz | (alias for SB1) | -- |
| https://sb-manazer.ra-energity.cz | (no auth) | sb-manager |

> Credentials are stored in `zadani/pristupy a servery/pgepilot_server.md` (private OneDrive).
> Passwords are currently hardcoded in `app.py` -- plan is to move to `/etc/energity/credentials.yaml`.

---

## Software Structure on SB1

```
/opt/energity/sb/
+-- device_controller/          # Modbus polling, device management
|   +-- device_service.py       # Flask :5010
|   +-- device_manager.py       # Device lifecycle, poll loop
|   +-- config_loader.py        # YAML -> DeviceConfig
|   +-- devices_config.yaml     # Active device configuration
|   +-- constraints_manager.py  # Command limits
|   +-- functions/              # Poll function YAML configs
|   +-- drivers/
|       +-- communication/      # modbus_tcp.py (thread-safe), modbus_rtu.py
|       +-- devices/            # goodwe_inverter.py, solax, victron, relay
|
+-- local_database/             # SQLite microservices
|   +-- sensor_data_service.py  # Flask :5011
|   +-- logs_data_service.py    # Flask :5012
|   +-- sensor_data.db
|
+-- web_interface/              # Local web UI
|   +-- app.py                  # Flask :5000
|   +-- config_api.py           # Flask :3000
|   +-- templates/              # login, user_section, service_section
|
+-- rpc_client/                 # Cloud communication
|   +-- main.py                 # FastAPI/RPC :3001
|
+-- communication_controller/   # Data flow orchestration
|   +-- smartbox_config_loader.py
|
+-- utils/
    +-- ntp_time_utils.py       # NTP time synchronization
```

---

## Known Issues

- `sb-test-3000` and `sb-test-3001` are in a failing loop on SB1 (legacy test services -- should be stopped/disabled)
- `sb-manager` on VPS has 168 historical restarts as of `2026-04-12 22:20 CEST`; restart counter remains a stability concern
- `energity-health` (:3007) and `energity-rpc-client` (:3001) bind to 0.0.0.0 (security risk)
- Passwords hardcoded in `app.py` (web interface)
- Modbus RTU driver lacks thread safety (only TCP is thread-safe)
- `modbus_reg_goodwe.yaml` has `scale: N/A` string on ~15 registers (47000, 47505, 47509, 47511, 47602, 47902-47905, 35185, 35187, 35189) -- causes `Invalid scale factor` warnings on every poll cycle. Driver handles gracefully but those register values are not stored. Affects both sb1 and sb4. Fix: replace `N/A` with numeric scale or remove those registers.
- `smartbox_service.py` logs `Error in smartboxPoll: 'NoneType' object has no attribute 'get'` every 30s on sb4 -- pre-existing bug in comm-controller when device-controller returns no data (0 enabled devices). Cosmetic on dev box, but would be real issue if it appears on production sb1.
- GoodWe `Serial=None, Model=None` at device init -- related to scale factor N/A bug above (serial/model registers have bad scale config)
- SmartBox provisioning still lacks strict UUID validation for `machine_id`. Live examples of bad propagated IDs: sb1 relay `b1bb905b-d285-50e3-98df-g5e62g1gc645` and sb7 inverter `c2cc916c-e396-51f4-a9f0-h6f73h2hd756`.
