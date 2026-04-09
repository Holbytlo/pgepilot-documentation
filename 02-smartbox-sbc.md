# 02 -- SmartBox / SBC Edge System

> Edge computing platform for local inverter monitoring and control via Raspberry Pi devices.
> Last updated: 2026-04-09

---

## Overview

SmartBox (SB) is an edge computing device (Raspberry Pi) installed at a customer's PV site. It communicates directly with inverters via Modbus TCP, stores data locally in SQLite, and syncs with PgePilot Cloud via JSON-RPC over a reverse SSH tunnel.

| Component | Description |
|-----------|-------------|
| **SmartBox (SB)** | Physical Raspberry Pi at customer site |
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

### Runtime Snapshot (verified 2026-04-09)

- `sb-manager`: clean `main@5fd115b`, PM2 online, 156 total restarts, current uptime 10 days
- `sb-router`: clean `main@4c4a7ad`, systemd active since 2026-03-12
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

### Git Push from VPS

```bash
GIT_SSH_COMMAND="ssh -i /home/limited/.ssh/githolbytlo -o IdentitiesOnly=yes" git push origin <branch>
```

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
| Git runtime | `/opt/energity/sb`, clean `devva@be59807` (verified 2026-04-09) |

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

> **Note (2026-04-04)**: Ports 20050-20055 are active on VPS -- this is a test SmartBox at a customer site. Not yet production, will be resolved gradually. Ask about current status before working on it.

### Services on SB1 (all systemd, root)

| Service | Port | Bind | Tech | Purpose |
|---------|------|------|------|---------|
| energity-device-controller | 5010 | 127.0.0.1 | Python/Flask | Modbus polling, REST API for devices |
| energity-sensor-db | 5011 | 127.0.0.1 | Python/Flask | SQLite sensor data storage |
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
  |     |     +-- goodwe_inverter.py  [ACTIVE]
  |     |     +-- solax_inverter.py   [READY]
  |     |     +-- victron_inverter.py [READY]
  |     +-- gpio_device.py (GPIO driver)
  |           +-- relay.py (relay driver)
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
- JWT authentication against auth.pgepilot.cz
- Methods: `smartboxSendData`, `smartboxPoll`
- Credentials configured in `rpc_client_config.yaml`

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
RPC Client (:3012) -- FastAPI, JWT auth, JSON-RPC 2.0
  | HTTPS to service.pgepilot.cz/rpc
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
|   +-- main.py                 # FastAPI/RPC :3012
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
- `sb-manager` on VPS has 156 historical restarts; it is currently online with a 10-day uptime, but the root cause is still unresolved
- `energity-health` (:3007) and `energity-rpc-client` (:3001) bind to 0.0.0.0 (security risk)
- Passwords hardcoded in `app.py` (web interface)
- Modbus RTU driver lacks thread safety (only TCP is thread-safe)
