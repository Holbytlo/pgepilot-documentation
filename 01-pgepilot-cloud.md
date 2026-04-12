# 01 -- PgePilot Cloud Platform

> Cloud monitoring and control platform for photovoltaic installations.
> Last updated: 2026-04-12

---

## Overview

PgePilot is a cloud-based monitoring platform that collects data from PV inverters (via vendor APIs and SmartBox edge devices), runs health checks, controls relays, generates forecasts, and serves dashboards.

| Property | Value |
|----------|-------|
| URL | pgepilot.cz |
| IP | 88.99.187.9 |
| Provider | Hetzner (ARM64) |
| Active plants | 35 collection points (17 GoodWe, 15 SolaX, 1 SmartBox); Task 29 collects 32 plants |
| Docker containers | 8 (verified 2026-04-10) |
| Databases | 8 (see [05-infrastructure.md](05-infrastructure.md)) |
| Primary API | `/api/v2/*` on port 8400 |
| Frontend | app.pgepilot.cz (Vue3, port 3060) |

---

## Docker Containers

| Container | Port | SSH Port | Tech | Purpose | Status |
|-----------|------|----------|------|---------|--------|
| **pgepilot_service** | 8400 | 2214 | PHP 8.1, Slim4 | API backend, TaskManager, task generation | Active |
| **pgepilot_worker1** | 6001 | 2261 | PHP 8.1 | Executes tasks (health check, backfill, relay) | Active |
| **pgepilot_worker2** | 6002 | 2262 | PHP 8.1 | Additional task worker | Active |
| **pgepilot_worker3** | 6003 | 2263 | PHP 8.1 | Additional task worker | Active |
| **pgepilot_jobmanager** | 5000-5001 | 2205 | Node.js v20, PM2 | Job orchestrator; current production recurring work is driven by DB-backed `task_definitions`, while the older `/scheduled_tasks` block stays commented on current `main` | Active |
| **pgepilot_servicedesk** | 3050, 3060 | 2206 | Vue3, Node.js | Frontend: servicedesk (:3050) + PGE App (:3060) | Active |
| **pgepilot_auth_srv** | 4000 | -- | Node.js, JWT | Legacy SmartBox / verify-token auth for non-migrated clients | Active |
| **nginx-proxy-manager** | 80, 443, 81 | -- | nginx | Reverse proxy, SSL, Let's Encrypt | Active |

> **Note**: `pgepilot_beapp` (legacy PHP backend) is NOT running. It exists as a Docker image but is not started.
> Workers 2 and 3 were added to distribute task execution load.
> **Auth note (2026-04-12)**: `pgepilot-service` now also exposes legacy-compatible SmartBox auth endpoints (`/login`, `/refresh-token`, `/verify-token`, `/issue-smartbox-token`). SB1, SB4, and SB7 currently use `service.pgepilot.cz`; `auth.pgepilot.cz` stays active for the remaining boxes.

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

### Task Definitions (verified 2026-04-09)

| ID | Name | Active | Notes |
|----|------|--------|-------|
| 15 | Plant Health Check | **Yes** | All plants, reads from cache |
| 17 | Control Relays | **Yes** | Per-plant relay strategy |
| 18 | Historical Data Backfill | **Yes** | Rewritten: GetStationHistoryDataChart (1min, 57 registers) -> canonical `pge_data.{code}_power_1m` with lineage. `power_bf` stays a reporting profile resolved over that history. |
| 19 | Energy Data Backfill (GoodWe kWh) | **Yes** | GoodWe reported kWh is aggregated into canonical `pge_data.{code}_energy_15m` with source lineage. |
| 20 | Sync Realtime | No | Migrated from crontab, remains disabled |
| 21 | Sync History | No | Migrated from crontab, remains disabled |
| 22 | Record Realtime to History | Yes | Reactivated on 2026-04-12; writes `realtime_state` into canonical `pge_data.{code}_power_1m` every minute |
| 23 | Compute Energy 15m | **Yes** | `every:900seconds`; prefers `power_1m`, falls back to `power_rt`, writes lineage into `energy_15m` |
| 24 | PV Forecast (forecast.solar) | No | `every:3600seconds`; endpoint exists in service, DB definition currently disabled |
| 25 | Load Forecast | No | `every:3600seconds`; endpoint exists in service, DB definition currently disabled |
| 26 | PV Forecast OpenMeteo | **Yes** | `every:3600seconds`; current active recurring forecast source |
| 27 | Forecast Correction | **Yes** | `daily:01:05`; adaptive correction task |
| 28 | Task Cleanup | **Yes** | `every:3600seconds`; task archive / maintenance |
| 29 | Collect Realtime Data | **Yes** | `every:300seconds`; fills cache + `realtime_state` + `power_rt` |
| 30 | OTE Spot Import Today | **Yes** | `daily:00:05`; imports published PT15M day-ahead prices for today |
| 31 | OTE Spot Import Tomorrow | **Yes** | `daily:12:10`; imports today + tomorrow, skipping tomorrow if OTE has not published yet |

> **Runtime note (2026-04-07)**: Current production scheduling is represented by `pgep_tasks.task_definitions` plus worker `/task` execution. The older `Holbytlo/pgepilot-js` `/scheduled_tasks` block is still commented on current `main`, so it is no longer the primary source of truth for recurring production jobs.

---

## Forecast System

Three forecast pipelines exist in the backend (`pv_forecast.php`, `load_forecast.php`, `forecast_correction.php`).
Current production uses DB-backed recurring task definitions for Open-Meteo and forecast correction; forecast.solar and load-forecast run endpoints remain available in service but their DB definitions are currently disabled.

### PV Forecast
- **Source**: forecast.solar Professional API
- **Granularity**: 15-minute intervals, 6 days ahead
- **Features**: Multi-string support (azimuth/tilt per array)
- **DB task**: `24`
- **Schedule**: Definition exists as `every:3600seconds`, currently disabled in DB
- **Trigger**: `POST /api/v2/tasks/run-pv-forecast`
- **Storage**: `pge_data.pv_forecast`
- **Config**: `pge_control.pv_forecast_config` (currently 2 plants enabled: VA, Kder Veltrusy)

### Load Forecast
- **Method**: Weekly profile from last 28 days + Czech holidays + seasonal correction + temperature correction
- **Seasonal factors**: January 1.35, July 0.70 (heating/cooling adjustments)
- **Granularity**: 15-minute intervals, 7 days ahead
- **DB task**: `25`
- **Schedule**: Definition exists as `every:3600seconds`, currently disabled in DB
- **Trigger**: `POST /api/v2/tasks/run-load-forecast`
- **Storage**: `pge_data.load_forecast`

### Weather Forecast
- **Source**: Open-Meteo API (temperature, cloudiness, GHI, precipitation, wind)
- **DB task**: `26`
- **Schedule**: `every:3600seconds`
- **Points**: 48 forecast points
- **Storage**: `pge_data.weather_forecast`

### Adaptive Correction
- **Method**: EMA (Exponential Moving Average), alpha=0.15
- **Compares**: Previous forecast vs actual production
- **Clamp range**: 0.50 -- 1.80
- **DB task**: `27`
- **Schedule**: `daily:01:05`
- **Trigger**: `POST /api/v2/tasks/run-forecast-correction`
- **Storage**: `pge_data.forecast_correction_log`

### Current Scheduling Model

Current production recurring execution is:

```text
task_definitions (MariaDB) -> service /run-tasks -> JobManager /add_task -> worker /task
```

The older `pgepilot-js` `/scheduled_tasks` and `/run_scheduled_task` handlers remain commented on current `main`, so they should be treated as legacy notes rather than the canonical production scheduler API.

```bash
# Run endpoints that do exist in current git source
curl -X POST http://pgepilot_service/api/v2/tasks/run-pv-forecast
curl -X POST http://pgepilot_service/api/v2/tasks/run-load-forecast
curl -X POST http://pgepilot_service/api/v2/tasks/run-forecast-correction
curl -X POST http://pgepilot_service/api/v2/tasks/run-pv-forecast-om
```

---

## OTE Day-Ahead Spot Import (added 2026-04-07)

PgePilot now imports OTE day-ahead electricity prices in **15-minute** resolution through a dedicated service route and recurring task definitions.

| Property | Value |
|----------|-------|
| Route | `GET /ote/import` |
| Alias | `GET /oteTest` |
| Default resolution | `PT15M` |
| Storage table | `ote_day_ahead_prices` |
| Import wrapper | `TaskController::runOteSpotImport()` |
| Active tasks | `30` (today), `31` (today + tomorrow) |

### Manual examples

```bash
curl 'http://pgepilot_service/ote/import'
curl 'http://pgepilot_service/ote/import?include_tomorrow=1'
curl 'http://pgepilot_service/ote/import?dry_run=1&include_tomorrow=1'
curl 'http://pgepilot_service/ote/import?date=2026-04-07&time_resolution=PT15M'
```

### Scheduling semantics

- Task `30` runs daily at `00:05` and refreshes today's published PT15M prices.
- Task `31` runs daily at `12:10` and fetches today plus tomorrow.
- If tomorrow's auction data has not been published yet, the importer marks that date as `skipped` instead of failing the whole job.

---

## Data Synchronization

### Sync mechanism (verified 2026-04-04)

Sync has been **migrated from container crontab to task_definitions** (IDs 20-22). The service container crontab no longer contains sync entries.

| Task ID | Name | Previously | Current mechanism |
|---------|------|------------|-------------------|
| 20 | Sync Realtime | `sync_realtime.php` (cron 1min) | task_definition (disabled) |
| 21 | Sync History | `sync_history.php` (cron 5min) | task_definition (disabled) |
| 22 | Record Realtime to History | `record_realtime_to_history.php` (cron 1min) | task_definition (active since 2026-04-12) |
| 23 | Compute Energy 15m | `compute_energy_15m.php` (cron 15min) | task_definition (**active**) |

The host crontab still runs:
- `mysql-backup.sh` (02:15 daily)
- `docker_healthcheck.sh` (every 5 min)
- `api_usage` cleanup (03:25 daily, DELETE > 30 days)

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
| SMARTBOX_RPC | Enabled | N/A | SB1 operational since 2026-04-04; plant VA_SB (ID 202) registered |

---

## Task 18 -- Historical Data Backfill (rewritten 2026-04-04)

Task 18 was rewritten to use a higher-resolution GoodWe API endpoint:

| Property | Old (before 2026-04-04) | New (after 2026-04-04) |
|----------|------------------------|------------------------|
| API method | GetPlantPowerChart (5min, 6 curves) | GetStationHistoryDataChart (1min, 57 registers) |
| Output table | power_bf only | canonical `power_1m`; customer `power_bf` is resolved as a reporting profile over canonical history |
| API call range | 1 call per day | 1 call per 7 days |
| API endpoint | SEMS Web: POST /api/HistoryData/GetStationHistoryDataChart | Same |
| Rate limit | -- | No rate limit observed on SEMS Web |
| Computed fields | None | pv_power_w = Vpv1*Ipv1 + Vpv2*Ipv2, battery_power_w = V*I, load = PV + Batt - Meter |

Additionally, `getPlantPowerChartAll()` now calls **GetChartByPlant** (chartIndexId=1, isDetailFull=true) which returns 12 curves (up from 6): includes per-phase meter/load and SOC2.

---

## Discovery Scripts (added 2026-04-04)

| Script | Purpose |
|--------|---------|
| `scripts/goodwe_discovery.php` | Sets backfill_start_date from conn_date, updates plant metadata (pvCapacity, batteryCapacity, classification, address) |
| `scripts/sems_fetch_all_plants.php` | Fetches all 228 SEMS plants to `sems_plant_discovery` table in pge_control |

---

## VA_SB -- SmartBox Pull Connector (added 2026-04-04)

New plant `VA_SB` (ID 202) registered as a SmartBox pull connector:

| Property | Value |
|----------|-------|
| Plant code | `va_sb` |
| Plant ID | 202 |
| Connector | SMARTBOX_RPC |
| Collection point | `va_sb` in pge_control |
| Time series tables | `va_sb_power_rt`, `va_sb_energy_15m` in pge_data |
| Health URL | https://sb1-health.ra-energity.cz/telemetry |

---

## Git Repositories

| Repo | Container | Branch | Contents |
|------|-----------|--------|----------|
| Holbytlo/pgepilot-service | `pgepilot_service` and workers 1/2/3 are now uniformly on `main@a921042` | main | PHP Slim4 API, TaskManager, DB migrations |
| Holbytlo/pgepilot-js | `pgepilot_jobmanager` in `/home/app`, clean `main@4346047`; recurring work is DB-backed via `task_definitions` | main | Node.js job orchestrator |
| Holbytlo/pgepilot-srv | `pgepilot_auth_srv` runtime is `/app` deploy artifact; no git metadata visible in container | main | Auth server (JWT, login) |
| Holbytlo/pge-app | `pgepilot_servicedesk:/home/app2/pge-app`, clean `main@3d7e6bb`; live bundle currently serves `/assets/index-lGjKNcFm.js` | main | Vue3 frontend |
| Holbytlo/servisdesk | `pgepilot_servicedesk:/home/app2/servicedesk`, clean `main@1c21bd4` | main | Admin UI |
| Holbytlo/sb | SB1/SB4/SB7 are now treated as rsync-deployed runtimes; audited key file hashes match local `devva@45a2fc0` | devva | Python SmartBox software |
| Holbytlo/sb-manager | VPS: `/opt/sb-manager`, clean `main@ef3261b`, PM2 online; historical restart counter is 168 as of 2026-04-12 evening | main | Node.js SB device management |
| Holbytlo/sb-router | VPS: `/opt/sb-router`, clean `main@4c4a7ad`, systemd active since 2026-03-12 | main | Host-based router for SB tunnel subdomains |

---

## Known Issues

- ~~GoodweSems OpenAPI token refresh broken~~ **FIXED** (2026-04-04) -- In-memory token cache + auto-retry on 100002 (commit `248351d`). All 32 plants now collected.
- GoodweSemsWeb credentials updated: old account [REDACTED] was deactivated (code 100029), new account [REDACTED] is working for both CrossLogin (web) and OpenAPI (GetToken)
- Legacy notes around `/scheduled_tasks` still appear in older docs; current production scheduling is DB-backed through `task_definitions`
- Workers were resynchronized with GitHub `main` on 2026-04-10, but future git-based redeploys still require temporary host-side SSH key injection because the containers do not keep GitHub credentials permanently
- Older docs say servicedesk `docker exec` fails; current reality is that `docker exec pgepilot_servicedesk sh -lc '...'` works
- Tailwind CSS loaded from CDN in PGE App (should be npm installed)
- `sb-manager` remains a stability concern: PM2 historical restart counter is now 168
- `task 18` is currently stabilized in production; last 6h snapshot on 2026-04-12 evening showed `1205 completed / 0 failed`
- `task 19` is also currently healthy; last 6h snapshot on the same evening showed `1215 completed / 9 sent / 0 failed`
- SB1, SB4, and SB7 no longer share one cloud identity; each box now has its own `login + smartbox_id + collection_point_id + device_id`
- SmartBox provisioning still lacks strict UUID validation for `machine_id`, and two bad IDs are already present in live config/data (`sb1 relay`, `sb7 inverter`)
