# 04 -- Mobile Application (Planned)

> React Native cross-platform app for PV monitoring. Currently in specification phase.
> Last updated: 2026-04-04

---

## Overview

The mobile app will share the same API v2 as the web app (PGE App), providing PV monitoring for technicians, project managers, and end customers on iOS and Android.

| Property | Value |
|----------|-------|
| Framework | React Native |
| Platforms | iOS + Android |
| Backend API | PgePilot API v2 (same as web) |
| Status | Specification phase |
| Estimated effort | 15-21 days |
| Source spec | `zadani/Tvorba mob app/Postup tvorby mobilni aplikace.md` |
| Entity model | `zadani/Tvorba mob app/ENTITY_MODEL_PGEPILOT_APP.md` |

---

## Target Users

| User type | Example | What they see |
|-----------|---------|---------------|
| Large institution | Municipality, hotel, energy community | Sharing dashboard, EANo/EANd overviews, manual override |
| Business (no sharing) | Manufacturing, commercial | SPOT prices, buy/sell optimization, production stats |
| Residential | House, small business | Production overview, boiler/wallbox automation, tips |

---

## Screen Structure

```
Tab 1: Domains (list of Control Domains)
  +-- Control Domain detail (= Plant view)
        +-- Power Flow diagram (PV/grid/battery/load from realtime_state)
        +-- Plant status (severity badge from v_plant_latest_status)
        +-- Control Points list (devices)
              +-- [tap BOILER]   -> Boiler screen
              +-- [tap WALLBOX]  -> Wallbox screen
              +-- [tap HEATING]  -> Heating screen
              +-- [tap POOL]     -> Pool screen

Tab 2: Charts
  +-- Plant -> Day/Week/Month/Year
  +-- Per Control Point (boiler kWh today etc.)

Tab 3: Alerts
  +-- event_log + plant_state_history (severity + timeline)

Tab 4: Overview (admin/service view)
  +-- All plants -- monitoring_status table
```

---

## Wireframes

8 HTML wireframes available in `zadani/Tvorba mob app/wireframes/`:

| File | Screen |
|------|--------|
| 01_login.html | Login screen |
| 02_dashboard.html | Dashboard with domain selector |
| 03_grafy.html | Charts (Day/Week/Month) |
| 04_instalace.html | Installation list |
| 05_detail_plant.html | Plant detail with power flow |
| 06_detail_machine.html | Machine detail (inverter) |
| 07_alerty.html | Alert list with severity filters |
| 10_domeny.html | Domain grid |
| index.html | Navigation index |

---

## Shared API v2

The mobile app uses the exact same API endpoints as the web app. See [06-api-reference.md](06-api-reference.md) for the full endpoint list.

Key endpoints:
- `POST /api/v2/auth/login` -- JWT authentication
- `GET /api/v2/dashboard` -- CP list with live data
- `GET /api/v2/domains` -- Domain list
- `GET /api/v2/collection-points/:code/live` -- Realtime power data
- `GET /api/v2/collection-points/:code/history` -- Historical data
- `GET /api/v2/collection-points/:code/forecast` -- PV/load forecast
- `POST /api/v2/commands` -- Control commands

---

## Development Approach

From the specification (`Postup tvorby mobilni aplikace.md`):

1. React Native + NestJS backend (or direct API v2)
2. Shared component library with web app patterns
3. Offline-first architecture for field technicians
4. Push notifications for alerts
5. Estimated timeline: 15-21 days for MVP

---

## Entity Model

The mobile entity model (from `ENTITY_MODEL_PGEPILOT_APP.md`, derived from live DB on 2026-03-28) follows the same structure as the web app:

```
User -> Customer -> Plant (code, EANd)
  Plant -> Machine[] (inverter, SN, live data)
  Plant -> Device[] (BOILER, WALLBOX, POOL, HEATING)
  Plant -> realtime_state (power flow)
  Plant -> event_log[] (alerts)
```

See [07-entity-model.md](07-entity-model.md) for the full domain model.
