# 08 -- Architecture Overview

> System-level architecture, data flows, component interactions, and development phases.
> Last updated: 2026-04-04

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          PRODUCTION (running)                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   pgepilot.cz                          ra-energity.cz                   │
│   ┌─────────────────────┐              ┌──────────────────┐             │
│   │  PgePilot Cloud     │              │  SmartBox VPS    │             │
│   │  ─────────────────  │              │  ──────────────  │             │
│   │  service (PHP)      │              │  sb-manager      │             │
│   │  worker1 (PHP)      │              │  sb-router       │             │
│   │  jobmanager (Node)  │              │  nginx           │             │
│   │  servicedesk (Vue3) │              └────────┬─────────┘             │
│   │  auth_srv (Node)    │                       │ SSH tunnel            │
│   │  nginx-proxy-mgr    │              ┌────────┴─────────┐             │
│   └────────┬────────────┘              │  SmartBox SB1    │             │
│            │                           │  (Raspberry Pi)  │             │
│    ┌───────┴───────┐                   │  ──────────────  │             │
│    │  MariaDB      │                   │  DeviceCtrl      │             │
│    │  ──────────── │                   │  LocalDB         │             │
│    │  pgepilot     │                   │  RPC Client      │             │
│    │  pge_control  │                   │  CommController   │             │
│    │  pge_data     │                   │  WebUI           │             │
│    │  pgepilot_data│                   └──────────────────┘             │
│    │  pgep_tasks   │                                                    │
│    └───────────────┘                                                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Component Interaction

### Task Execution Loop (every 3 seconds)

```
JobManager (Node.js, port 5000)
  |
  | GET /run-tasks (every 3s)
  v
Service (PHP, port 8400)
  |
  | TaskManager.generateTasks()
  |   -> 1 task per plant per active task_definition
  |   -> respects connector_config (realtime_enabled, backfill_enabled)
  |
  | TaskManager.sendPendingTasks()
  |   -> POST /add_task to JobManager
  v
JobManager
  |
  | POST /task to Worker1
  v
Worker1 (PHP, port 6001)
  |
  | Executes task:
  |   - Health Check: calls GoodWe/SolaX API (reads from cache)
  |   - Collect Data: calls API, stores in power_rt + realtime_state
  |   - Control Relays: evaluates strategy, toggles Supla relays
  |   - Compute Energy: aggregates power_rt -> energy_15m
  v
Result -> stored in pgepilot/pge_control/pge_data databases
```

### Forecast Scheduling (via JobManager)

```
JobManager scheduler (setInterval 60s)
  |
  | checks time:
  |   :17 -> POST /api/v2/tasks/run-pv-forecast
  |   :23 -> POST /api/v2/tasks/run-load-forecast
  |   1:05 -> POST /api/v2/tasks/run-forecast-correction
  v
Service -> runs PHP forecast scripts -> results to pge_data tables
```

### SmartBox Data Flow

```
GoodWe Inverter (Modbus TCP, 192.168.0.70:502)
  |
  | poll every 5-10s (pymodbus)
  v
Device Controller (:5010)
  |
  | POST /sensor-data
  v
Sensor DB (:5011, SQLite)
  |
  | GET /sensor-data/range
  v
Communication Controller
  |
  | smartboxSendData (every 5s)
  v
RPC Client (:3012, FastAPI)
  |
  | JSON-RPC 2.0 over HTTPS
  v
PgePilot Cloud (service.pgepilot.cz/rpc)
  |
  | Stores in rpc_kv -> processed to pge_control/pge_data
```

### Backfill Data Flow (rewritten 2026-04-04)

Task 18 now uses a higher-resolution API endpoint that retrieves 1-minute register data:

```
Task 18: Historical Data Backfill (per GoodWe plant)
  |
  | GoodweSemsWeb.getStationHistoryData()
  |   POST /api/HistoryData/GetStationHistoryDataChart
  |   (SEMS Web, 1 call per 7-day range, no rate limit)
  v
Raw response: 57 registers at 1-minute resolution
  |
  | Parse + compute derived fields:
  |   pv_power_w  = Vpv1*Ipv1 + Vpv2*Ipv2
  |   battery_power_w = V * I
  |   load = PV + Batt - Meter
  v
INSERT INTO {code}_power_1m (pge_data)
  |
  | Aggregate 1min -> 5min
  v
INSERT INTO {code}_power_bf (pge_data, includes battery_soc2_pct)

Additionally:
  getPlantPowerChartAll() -> GetChartByPlant (12 curves)
    -> per-phase meter/load + SOC2
    -> power_bf tables
```

### Discovery Flow (added 2026-04-04)

```
scripts/sems_fetch_all_plants.php
  |
  | GoodweSemsWeb.getPowerStationList()
  |   POST /api/PowerStationMonitor/QueryPowerStationMonitor
  |   (paginated, returns 228 SEMS plants)
  v
sems_plant_discovery (pge_control)
  |
  | in_pgepilot flag marks registered plants
  v
scripts/goodwe_discovery.php
  |
  | GoodweSemsWeb.getPlantMonitor() + getInventers()
  |   -> backfill_start_date from conn_date
  |   -> pvCapacity, batteryCapacity, classification, address
  v
Updates cp_collection_points + cp_machines metadata
```

---

## One API, Three Clients

```
                PgePilot API v2
                (pge_control + pge_data)
                     │
          ┌──────────┼──────────┐
          │          │          │
     PGE App     Mobile App  Energity Web
     (C1, done)  (C2, plan)  (C3, future)
     Vue3+Vite   React Native  External team
     app.pge..                  energity.cz
```

All three clients use the SAME API endpoints. Energity Web may add onboarding endpoints.

---

## Development Phases

### Phase A: SmartBox Control (sb.* TOU API) -- IN PROGRESS

TOU (Time-of-Use) engine on SmartBox:

```
PgePilot Cloud
  |
  | sb.setDefaults, sb.setTouSchedule, sb.queueCommand
  v
SmartBox TOU Engine
  |
  | defaults -> TOU schedule -> overrides -> direct commands
  |
  | desired -> effective (after HW constraints) -> southbound
  v
Physical devices (GoodWe Modbus, Supla relays)
```

- v0.1 = PLANT device (inverter) -- in progress
- v0.2 = + BOILER, EVSE, POOL, HVAC

### Phase B: New Database (pge_control + pge_data) -- DONE

- pge_control: 18+ tables (entities, commands, events, configs)
- pge_data: 81 tables (per-plant time series + forecast)
- Backfill from legacy completed
- Realtime sync running (cron 1min + 5min)
- Legacy NOT disabled (coexistence)

### Phase C: API + Clients -- C1 DONE

- C1: PGE App (Vue3 web) -- PRODUCTION
- C2: Mobile App (React Native) -- PLANNED
- C3: Energity Web (external team) -- FUTURE

### Phase D: Advanced Features -- FUTURE

- D1: Forecast improvements (already partially done: PV + load + weather)
- D2: Planned control (dispatcher, SPOT price optimization)
- D3: Energy communities (EANd/EANo sharing, virtual battery)
- D4: SB plugin framework (sb-core + plugin drivers, OTA updater)
- D5: Additional device types (BOILER TOU, EVSE, POOL, HVAC)

---

## Legacy Coexistence Strategy

```
LEGACY (running, NOT disabled):       NEW (running alongside):
  pgepilot DB                            pge_control DB
    plants ·····backfill····>              cp_collection_points
    machines ···backfill····>              cp_machines
    relay_groups ···········>              cp_devices
    event_log ··············>              cp_events (empty)

  pgepilot_data DB                       pge_data DB
    {code}_power_5m ········>              {code}_power_1m (57 registers, 1min)
    {code}_energy_5m ·······>              {code}_energy_15m
                                           {code}_power_bf (aggregated from 1m)

  Legacy API v1 (still running)          API v2 (primary)
```

- Sync runs continuously (1min realtime, 5min history)
- Legacy will be disabled only when new system fully replaces all functionality
- No ALTER TABLE on legacy tables -- all changes go to pge_control/pge_data
