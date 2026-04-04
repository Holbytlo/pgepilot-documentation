# 01 -- PgePilot Cloud Platform

> Cloud monitoring and control platform for photovoltaic installations.
> Last updated: 2026-04-04

---

## Overview

PgePilot is a cloud-based monitoring platform that collects data from PV inverters (via vendor APIs and SmartBox edge devices), runs health checks, controls relays, generates forecasts, and serves dashboards.

| Property | Value |
|----------|-------|
| URL | pgepilot.cz |
| IP | 88.99.187.9 |
| Provider | Hetzner (ARM64) |
| Active plants | 23 (7 GoodWe, 15 SolaX, 1 SmartBox) |
| Docker containers | 7 |
| Databases | 8 (see [05-infrastructure.md](05-infrastructure.md)) |
| Primary API | `/api/v2/*` on port 8400 |
| Frontend | app.pgepilot.cz (Vue3, port 3060) |

---

## Docker Containers

| Container | Port | Tech | Purpose | Status |
|-----------|------|------|---------|--------|
| **pgepilot_service** | 8400 | PHP 8.1, Slim4 | API backend, TaskManager, task generation | Active |
| **pgepilot_worker1** | 6001 | PHP 8.1 | Executes tasks (health check, backfill, relay control) | Active |
| **pgepilot_jobmanager** | 5000 | Node.js v20, PM2 | Job orchestrator + scheduler (forecast tasks) | Active |
| **pgepilot_servicedesk** | 3050, 3060 | Vue3, Node.js | Frontend: servicedesk (:3050) + PGE App (:3060) | Active |
| **pgepilot_auth_srv** | 4000 | Node.js, JWT | Authentication (login, token validation) | Active |
| **pgepilot_beapp** | 8001 | PHP 8.1 | LEGACY -- do not use | Legacy |
| **nginx-proxy-manager** | 80, 443, 81 | nginx | Reverse proxy, SSL, Let's Encrypt | Active |

---

## Task System

### How Tasks Work

```
JobManager (Node.js) --every 3s--> Service /run-tasks
Service --> TaskManager.generateTasks() --> 1 task per plant per definition
Service --> TaskManager.sendPendingTasks() --> POST /add_task to JobManager
JobManager --> POST /task to Worker1
Worker1 --> calls GoodWe/SolaX API --> stores in DB
```

### Active Task Definitions

| ID | Name | Function | Interval | Notes |
|----|------|----------|----------|-------|
| 15 | Plant Health Check | checkPlantHealth | 5 min | 32 plants (17 GW + 15 SolaX), reads from cache |
| 17 | Control Relays | controlRelays | 5 min | Per-plant relay strategy |
| 23 | Compute Energy 15m | computeEnergy15m | 15 min | Aggregates power_rt to energy_15m |
| 29 | Collect Realtime Data | collectRealtimeData | per cycle | GW + SolaX, fills cache + realtime_state + power_rt |

### Disabled Tasks

| ID | Name | Why |
|----|------|-----|
| 18 | Historical Data Backfill | `tpl_power_bf` table missing -- needs CREATE TABLE |
| 19 | Energy Data Backfill | Disabled during debugging, can be re-enabled (GoodWe only) |

---

## Forecast System

Three independent forecast pipelines, all managed by the JobManager scheduler (not crontab).

### PV Forecast
- **Source**: forecast.solar Professional API
- **Granularity**: 15-minute intervals, 6 days ahead
- **Features**: Multi-string support (azimuth/tilt per array)
- **Schedule**: Every hour at :17
- **Trigger**: `POST /api/v2/tasks/run-pv-forecast`
- **Storage**: `pge_data.pv_forecast`
- **Config**: `pge_control.pv_forecast_config` (currently 2 plants enabled: VA, Kder Veltrusy)

### Load Forecast
- **Method**: Weekly profile from last 28 days + Czech holidays + seasonal correction + temperature correction
- **Seasonal factors**: January 1.35, July 0.70 (heating/cooling adjustments)
- **Granularity**: 15-minute intervals, 7 days ahead
- **Schedule**: Every hour at :23
- **Trigger**: `POST /api/v2/tasks/run-load-forecast`
- **Storage**: `pge_data.load_forecast`

### Weather Forecast
- **Source**: Open-Meteo API (temperature, cloudiness, GHI, precipitation, wind)
- **Points**: 48 forecast points
- **Storage**: `pge_data.weather_forecast`

### Adaptive Correction
- **Method**: EMA (Exponential Moving Average), alpha=0.15
- **Compares**: Previous forecast vs actual production
- **Clamp range**: 0.50 -- 1.80
- **Schedule**: Daily at 1:05
- **Trigger**: `POST /api/v2/tasks/run-forecast-correction`
- **Storage**: `pge_data.forecast_correction_log`

### JobManager Scheduler API

```bash
# Check scheduled tasks status
curl http://pgepilot_jobmanager:5000/scheduled_tasks

# Manually trigger a forecast
curl -X POST http://pgepilot_jobmanager:5000/run_scheduled_task -d '{"name":"pv_forecast"}'
```

---

## Data Synchronization

Sync crons run inside the `pgepilot_service` container crontab:

| Cron | Script | Interval | Purpose |
|------|--------|----------|---------|
| `* * * * *` | sync_realtime.php | 1 min | Legacy plants -> realtime_state (pge_control) |
| `*/5 * * * *` | sync_history.php | 5 min | Legacy history -> pge_data time series |
| `* * * * *` | record_realtime_to_history.php | 1 min | Realtime snapshot -> power_rt tables |
| `*/15 * * * *` | compute_energy_15m.php | 15 min | Aggregate power_rt -> energy_15m |

**Note**: Plan is to migrate all sync crons into the JobManager scheduler (Priority 1.4 in roadmap).

---

## Connector Self-Governance (added 2026-04-04)

Each API connector (GoodWe, SolaX) has budget and cache awareness via `ConnectorBudgetTrait.php`:

- **`isEnabled(taskType)`** -- reads `connector_config.realtime_enabled` / `backfill_enabled`
- **`canAfford(taskType)`** -- checks API rate limiter budget
- **`getCached(key)` / `setCache(key, data, ttl)`** -- reads/writes `api_response_cache`
- **Task generation** respects connector flags: HC skips plants where `realtime_enabled = 0`

### Connector Status

| Connector | Realtime | Backfill | Notes |
|-----------|----------|----------|-------|
| GOODWE_SEMS | Enabled | Enabled | 4 new cache methods not yet deployed to git |
| SOLAX_CLOUD | Enabled | **Disabled** | API limit protection (10K/day, 100/min) |
| SMARTBOX_RPC | Enabled | N/A | SB1 operational since 2026-04-04 |

---

## Git Repositories

| Repo | Container | Branch | Contents |
|------|-----------|--------|----------|
| Holbytlo/pgepilot-service | service + worker1 | main | PHP Slim4 API, TaskManager, DB migrations |
| Holbytlo/pgepilot-js | jobmanager | main | Node.js job orchestrator, scheduler |
| Holbytlo/pgepilot-srv | auth_srv | main | Auth server (JWT, login) |
| Holbytlo/pge-app | servicedesk (PM2 pge-app) | main | Vue3 frontend |
| Holbytlo/sb | SB1: /opt/energity/sb | devva | Python SmartBox software |
| Holbytlo/sb-manager | VPS: /opt/sb-manager | main | Node.js SB device management |

---

## Known Issues

- GoodWe cache methods (4 new) are local edits in worker1, **not committed to git** (session ended before deploy on 2026-04-04)
- Sync crons still in service container crontab instead of JobManager scheduler
- Worker deploy uses `docker cp` (no functional git pull inside worker1)
- Servicedesk `docker exec` does not work (OCI error) -- must use `nsenter`
- Tailwind CSS loaded from CDN in PGE App (should be npm installed)
- `sb-manager` on VPS has 155 restarts in 4 days (stability issue)
