# 03 -- PGE App (Web Frontend)

> Vue 3 single-page application for PV installation monitoring at app.pgepilot.cz.
> Last updated: 2026-04-05

---

## Overview

PGE App is a client-facing Vue 3 SPA for monitoring PV installations. It runs alongside the servicedesk admin panel inside the same Docker container.

| Property | Value |
|----------|-------|
| URL | app.pgepilot.cz |
| Port | 3060 |
| PM2 process | pge-app (id 2) |
| Container | pgepilot_servicedesk |
| Git repo | Holbytlo/pge-app (branch: main) |
| Tech stack | Vue 3 + TypeScript + Vite 8 + Tailwind CSS (CDN) + Chart.js 4 (CDN) |

There are two apps in the servicedesk container:

| App | PM2 | Port | Domain | Directory |
|-----|-----|------|--------|-----------|
| Servicedesk (admin) | servicedesk (id 0) | 3050 | sd.pgepilot.cz | /home/app2/servicedesk/ |
| PGE App (client) | pge-app (id 2) | 3060 | app.pgepilot.cz | /home/app2/pge-app/ |

---

## Routing

| Path | Component | Access | Description |
|------|-----------|--------|-------------|
| `/login` | Login.vue | Public | JWT login |
| `/` | Dashboard.vue | Auth | Main overview: domains + KPI cards + CP table |
| `/cp/:code` | CPDetail.vue | Auth | Plant detail: live power, charts, devices, controls |
| `/domeny` | Domains.vue | Auth | Domain grid with CP tags |
| `/grafy` | Grafy.vue | Auth | Analytical charts with CP selector |
| `/instalace` | Instalace.vue | Auth | Installation list with connector, capacity, device count, status |
| `/alerty` | Alerty.vue | admin, operator | Alerts: critical/warning/stale filters |
| `/uzivatele` | Users.vue | admin | User management + RBAC grants |
| `/nastaveni` | Nastaveni.vue | Auth | User profile, default domain, notification preferences |

---

## Key Components

### Dashboard.vue
- Domain selector (filters on Control Domain)
- KPI cards (total power, consumption, feed-in)
- CP table with live data (sorted by severity)

### CPDetail.vue
- Live power cards (PV, Load, Grid, Battery, SoC)
- PowerFlowCard -- energy flow visualization (PV -> House -> Grid/Battery)
- EnergyChart -- shared chart with Day/Week/Month/Year tabs
- Device list (devices + machines)
- Control panel -- setExportLimit, setBatteryTarget commands

### EnergyChart.vue (shared component)
- Day/Week/Month/Year tabs
- W/kWh toggle
- Per-series visibility (PV, Load, Grid, Battery, SoC)
- Dual-range time slider (Day tab)
- Aggregation: 5min, 1h, per-day, per-week, per-month
- Energy summary bars

### PowerFlowCard.vue
- Animated energy flow between PV, house, grid, and battery
- Real-time values from API

### Instalace.vue
- List of collection points with code, label, connector type, PV/battery capacity, device count, status
- Click-through from installation row to CP detail
- Uses `/dashboard` as its source

### Nastaveni.vue
- Profile summary (name/login/role/email)
- Default domain selection via `/users/me/preferences`
- Notification/display preferences persisted through the same preferences endpoint

---

## Multi-Tenant Theming

Current `main` does not implement hostname-based theme switching in `main.ts`.
Branding and navigation shell are defined directly in `App.vue`.

---

## API Integration

All API calls go through `/api/v2/*` (proxied to `pgepilot.cz:8400` by `server.cjs`).

| Endpoint | Method | Used by |
|----------|--------|---------|
| `/dashboard` | GET | Dashboard.vue |
| `/domains` | GET | Dashboard.vue, Domains.vue |
| `/collection-points/:code` | GET | CPDetail.vue |
| `/collection-points/:code/live` | GET | CPDetail.vue |
| `/collection-points/:code/history` | GET | CPDetail.vue, Grafy.vue |
| `/collection-points/:code/energy-summary` | GET | CPDetail.vue |
| `/collection-points/:code/relay-groups` | GET | CPDetail.vue |
| `/commands` | POST | CPDetail.vue |
| `/users` | GET/POST | Users.vue |
| `/users/:id/grants` | GET/POST/DELETE | Users.vue |
| `/users/me/preferences` | GET/PUT | Nastaveni.vue |
| `/auth/login` | POST | Login.vue |

Backend forecast endpoint `/collection-points/:code/forecast` exists in `pgepilot-service`,
but the current `pge-app` `main` branch does not expose a dedicated `/predikce` route.

Auth: JWT token stored in localStorage, sent as `Authorization: Bearer <token>`.

---

## Build and Deploy

```bash
# Development (Vite dev server with proxy)
cd /home/app2/pge-app && npm run dev

# Build for production
npm run build   # -> dist/

# Restart in production (PM2)
pm2 restart pge-app
```

### Deploy to container

```bash
# Copy files to servicedesk container
docker cp /tmp/Component.vue pgepilot_servicedesk:/home/app2/pge-app/src/views/...

# Build inside container (via nsenter, docker exec doesn't work)
SD_PID=$(docker inspect pgepilot_servicedesk --format '{{.State.Pid}}')
nsenter -t $SD_PID -m -u -i -n -p -- bash -c 'cd /home/app2/pge-app && npm run build'
```

---

## File Structure

```
/home/app2/pge-app/
+-- server.cjs              # HTTP server (port 3060, proxy /api/* + static)
+-- index.html              # Vite dev entry
+-- package.json
+-- vite.config.ts
+-- dist/                   # Built output
|   +-- index.html
|   +-- assets/
+-- src/
    +-- main.ts             # Entry + theme detection
    +-- App.vue             # Shell (sidebar + topbar + router-view)
    +-- router.ts           # 9 routes
    +-- api.ts              # Axios client (/api/v2/*)
    +-- views/
    |   +-- Login.vue
    |   +-- Dashboard.vue
    |   +-- CPDetail.vue
    |   +-- Domains.vue
    |   +-- Grafy.vue
    |   +-- Alerty.vue
    |   +-- Users.vue
    |   +-- Instalace.vue
    |   +-- Nastaveni.vue
    +-- components/
        +-- EnergyChart.vue
        +-- PowerFlowCard.vue
        +-- HistoryChart.vue
```

---

## Known Issues

- Tailwind CSS loaded from CDN (should use `npm install tailwindcss` for production)
- Chart.js also from CDN
- `docker exec` doesn't work for servicedesk container (OCI error) -- must use `nsenter`
- Older notes mention `/predikce` and `Forecast.vue`, but that route/component is not present on current `main`
