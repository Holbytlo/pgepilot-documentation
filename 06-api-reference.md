# 06 -- API Reference

> All API endpoints: PgePilot v2, legacy v1, Email, RPC, JobManager, and external connectors.
> Last updated: 2026-04-12

---

## Authentication

### JWT Login
```
POST /api/v2/auth/login
Content-Type: application/json

{ "email": "user@example.com", "password": "..." }
```

Response: `{ "token": "eyJ...", "refreshToken": "..." }`

Use the token in all subsequent requests:
```
Authorization: Bearer <token>
```

Current production login goes through `pgepilot-service` `POST /api/v2/auth/login`, which issues the JWT used by `pge-app`.
`pgepilot_auth_srv` still exists as a parallel legacy auth component, but it is not the only auth path in the stack anymore.

---

## API v2 Endpoints (36+)

Base URL: `https://api.pgepilot.cz/api/v2/` (or `pgepilot.cz:8400/api/v2/` internally)

Implemented primarily in: `pgepilot-service/app/routes_pge_control.php`

### Dashboard & Navigation

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/dashboard` | GET | CP list with live data (filtered by domain) |
| `/domains` | GET | List of domains for current user |
| `/summary` | GET | Aggregate statistics |
| `/health` | GET | Health check endpoint |

### Collection Points

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/collection-points` | GET | All collection points |
| `/collection-points/{code}` | GET | CP detail (devices, machines, connectors, source options, history usage options) |
| `/collection-points/{code}/live` | GET | Realtime power data from realtime_state |
| `/collection-points/{code}/history` | GET | Historical/reporting data. Params: `dataset`, `usage`, `source`, `from`, `to` |
| `/collection-points/{code}/energy-summary` | GET | Energy summary (kWh totals). Params: `usage`, `source`, `from`, `to` |
| `/collection-points/{code}/devices` | GET | Device list |
| `/collection-points/{code}/relay-groups` | GET | Relay groups (control points) |
| `/collection-points/{code}/battery-mode` | GET | Current battery mode |
| `/collection-points/{code}/battery-mode` | PUT | Set battery mode |
| `/collection-points/{code}/commands` | GET | Command history |
| `/collection-points/{code}/forecast` | GET | PV + load + weather forecast + comparison + history |

### Commands

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/commands` | POST | Create control command (setExportLimit, setBatteryTarget, etc.) |

### Alerts

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/alerts` | GET | Alert list (filterable by severity, status) |

### Users & RBAC

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/users` | GET | List all users |
| `/users` | POST | Create user |
| `/users/{id}/grants` | GET | User's RBAC grants |
| `/users/{id}/grants` | POST | Add grant |
| `/users/{id}/grants` | DELETE | Remove grant |
| `/users/me/preferences` | GET | Current user preferences (selected domain) |
| `/users/me/preferences` | PUT | Update preferences |

### Forecast Tasks

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/tasks/run-pv-forecast` | POST | Trigger PV forecast (called by JobManager) |
| `/tasks/run-load-forecast` | POST | Trigger load forecast |
| `/tasks/run-forecast-correction` | POST | Trigger adaptive correction |
| `/tasks/run-pv-forecast-om` | POST | Trigger Open-Meteo PV forecast |

### Onboarding

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/onboarding/claim/smartbox` | POST | Register SmartBox device |
| `/onboarding/claim/vendor` | POST | Register vendor device |

### Additional Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/current-cp-status` | GET | Current status of all CPs |
| `/recommended-start` | GET | Recommended start times for appliances |
| `/indicative-savings` | GET | Indicative savings estimate |
| `/pv-energy-targets` | GET | PV energy targets |
| `/battery-energy-targets` | GET | Battery energy targets |
| `/overflows` | GET | Grid overflow data |
| `/community-overflows` | GET | Community overflow data |

---

## Legacy API v1

Still running on the same service container. Used by older clients.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/login` | POST | Legacy login path; mixed transitional auth flow, not the primary `pge-app` login anymore |
| `/UserByLogin` | GET | User info |
| `/getCustomerPlants` | GET | Plant IDs for customer |
| `/getPlantInfo` | GET | Plant detail |
| `/getPlantEnergyData` | GET | Energy history |

---

## History and Reporting Contract

`/collection-points/{code}` now exposes both source and reporting defaults:

- `source_options`
- `history_usage_options`
- `default_source`
- `default_history_dataset`
- `default_history_usage`

History is resolved by a combination of:

- `dataset` = requested aggregation or dataset family (`power_bf`, `power_1m`, `power_rt`, `power_1h`, `power_1d`, `energy_15m`)
- `usage` = use-case profile that tells backend policy what the caller wants
- `source` = effective or connector-specific source selection

Important runtime note:

- `power_1m` is the canonical history store in `pge_data`
- `power_bf` is the default customer-facing reporting profile
- when no dedicated `{code}_power_bf` table exists, backend resolves `power_bf` by aggregating canonical `power_1m`
- SmartBox push (`smartboxSendData`) and GoodWe backfill now both feed the same canonical history/energy layer in `pge_data`

Supported `usage` values in production:

- `customer_history_power` -- default power history for app/web/mobile
- `customer_history_energy` -- default energy reporting profile
- `forecast_input_power` -- preferred input profile for forecast/analytics

Client rule of thumb:

- customer reporting screens pass explicit aggregation dataset plus `usage=customer_history_power`
- energy summary calls use `usage=customer_history_energy`
- forecast or analytics jobs use `usage=forecast_input_power`

Important response metadata returned by `/history` and `/energy-summary`:

- `source_requested`
- `source_selected`
- `usage_requested`
- `usage_selected`
- `dataset`
- `source_options`
- `usage_options`

---

## Email API

See [11-email-api.md](11-email-api.md) for full documentation.

```
POST https://service.pgepilot.cz/sendEmail
```

---

## OTE Market Data Import

Root-level service route (not under `/api/v2`):

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/ote/import` | GET | Import OTE day-ahead PT15M/PT60M prices into `ote_day_ahead_prices` |
| `/oteTest` | GET | Alias to the same importer |

Supported query params:

- `date=YYYY-MM-DD`
- `date_from=YYYY-MM-DD`
- `date_to=YYYY-MM-DD`
- `include_tomorrow=1`
- `time_resolution=PT15M|PT60M`
- `dry_run=1`
- `allow_missing_future=1`
- `table=ote_day_ahead_prices`

---

## SmartBox RPC API (sb.* namespace)

JSON-RPC 2.0 over HTTPS. Used for SmartBox TOU control.

Endpoint: `POST https://service.pgepilot.cz/rpc`

| Method | Purpose | Parameters |
|--------|---------|------------|
| `sb.setDefaults` | Set baseline parameters | mode, exportLimitW, gridTargetW, batteryTargetW |
| `sb.setTouSchedule` | Set recurring time rules | months, days, hours -> parameters |
| `sb.setDateTimeOverrides` | Set specific date+time override | datetime, parameters |
| `sb.queueCommand` | Queue direct command | setMode, setExportLimit, setGridTarget, setBatteryTarget, clearTargets |
| `sb.getState` | Get device state | Returns actual / desired / effective |
| `sb.getTouPlan` | Get current TOU plan | -- |
| `sb.evaluateTou` | Evaluate TOU at time | datetime |
| `sb.listCommands` | List pending commands | -- |
| `sb.listEvents` | List device events | -- |

SmartBox communication methods:
| Method | Direction | Purpose |
|--------|-----------|---------|
| `smartboxSendData` | SB -> Cloud | Push telemetry data |
| `smartboxPoll` | SB -> Cloud | Poll for pending commands/config |

---

## Task Orchestration (internal)

Current production recurring execution is DB-backed and uses these internal endpoints:

| Component | Endpoint | Method | Description |
|----------|----------|--------|-------------|
| service | `/run-tasks` | GET | Generate DB tasks from active definitions and send pending jobs to JobManager |
| jobmanager | `/add_task` | POST | Accept queued task from service |
| worker | `/task` | POST | Execute a task payload by calling the matching `TaskController` function |

> **Legacy note**: older docs referenced JobManager `/scheduled_tasks` and `/run_scheduled_task`, but those handlers are commented on current `Holbytlo/pgepilot-js` `main` and are no longer the primary production scheduler model.

---

## External API Connectors

### GoodWe OpenAPI
- Base: `https://eu.openapi.semsportal.com`
- Auth: MD5 hash login -> 2-hour token
- Rate: 3600 requests/hour
- Used for: Plant data, inverter details, energy history

### GoodWe Web API
- Base: `https://eu.semsportal.com/api`
- Auth: CrossLogin -> base64 token
- Used for: Alternative data access

### SolaxCloud
- Base: `https://global.solaxcloud.com/api/v2`
- Auth: tokenId in header
- Rate: 10,000/day, 100/min
- Used for: SolaX inverter data
- **Backfill disabled** (API limit protection)

### forecast.solar
- Tier: Professional
- Used for: PV production forecast, multi-string support
- Config stored in: `pge_control.pv_forecast_config`

### Open-Meteo
- Base: `https://api.open-meteo.com`
- Auth: None (free tier)
- Used for: Weather forecast (temperature, cloudiness, GHI, precipitation, wind)
