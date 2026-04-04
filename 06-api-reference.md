# 06 -- API Reference

> All API endpoints: PgePilot v2, legacy v1, Email, RPC, JobManager, and external connectors.
> Last updated: 2026-04-04

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

Token is issued by `pgepilot_auth_srv` (port 4000).

---

## API v2 Endpoints (36+)

Base URL: `https://api.pgepilot.cz/api/v2/` (or `pgepilot.cz:8400/api/v2/` internally)

Implemented in: `pgepilot-service/src/Routes/routes_pge_control.php`

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
| `/collection-points/{code}` | GET | CP detail (devices, machines, connectors) |
| `/collection-points/{code}/live` | GET | Realtime power data from realtime_state |
| `/collection-points/{code}/history` | GET | Historical data. Params: `dataset` (power_1m, energy_15m), `from`, `to` |
| `/collection-points/{code}/energy-summary` | GET | Energy summary (kWh totals) |
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
| `/login` | POST | Auth via auth_srv:4000 -> JWT |
| `/UserByLogin` | GET | User info |
| `/getCustomerPlants` | GET | Plant IDs for customer |
| `/getPlantInfo` | GET | Plant detail |
| `/getPlantEnergyData` | GET | Energy history |

---

## Email API

See [11-email-api.md](11-email-api.md) for full documentation.

```
POST https://service.pgepilot.cz/sendEmail
```

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

## JobManager API

Base URL: `http://pgepilot_jobmanager:5000/` (internal only)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/scheduled_tasks` | GET | List all scheduled tasks with status |
| `/run_scheduled_task` | POST | Manually trigger a scheduled task. Body: `{"name":"pv_forecast"}` |
| `/add_task` | POST | Add task to queue (called by service) |
| `/task` | POST | Execute task (called by JobManager -> worker1) |

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
