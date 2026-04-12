# 04 -- Mobile Application

> React Native cross-platform app for PGE Pilot on iOS and Android.
> Last updated: 2026-04-11

---

## Overview

The mobile app is no longer only a specification. There is an active implementation in the `pge-mobile` repository on GitHub and in this local workspace.

| Property | Value |
|----------|-------|
| Framework | React Native 0.84 + TypeScript |
| Platforms | iOS + Android |
| Backend API | PgePilot API v2 (shared with web) |
| Status | Implemented MVP, active development |
| Git repo | Holbytlo/pge-mobile (`main` currently at `037f5fb`) |
| Package id | `cz.pgepilot.mobile` |
| App name | `PGE Pilot` |
| Session storage | AsyncStorage |

The app already provides the shared customer monitoring surface used in PGE App:

- login
- `Přehled`
- `Domény`
- `Grafy`
- `Instalace`
- `Alerty`
- collection point detail
- profile / selected domain context

---

## Navigation

Current navigation in `src/navigation/RootNavigator.tsx`:

| Surface | Screen | Access | Notes |
|---------|--------|--------|-------|
| Login | `LoginScreen` | Public | JWT login against API v2 |
| Přehled | `DashboardScreen` | Auth | Domain chips, KPI cards, collection point list |
| Domény | `DomainsScreen` | Auth | Aggregated domain inventory with quick access to installations |
| Instalace | `InstallationsScreen` | Auth | Cross-domain installation list with domain filter |
| Detail lokality | `CollectionPointDetailScreen` | Auth | Live telemetry, charts, commands |
| Grafy | `ChartsScreen` | Auth | Day/Week/Month/Year charts, source selector |
| Alerty | `AlertsScreen` | admin, operator | Matches web role gating |
| Profil | `ProfileScreen` | Auth | User context, selected domain, logout |

Deep links:

- `pgepilot://login`
- `pgepilot://dashboard`
- `pgepilot://charts`
- `pgepilot://alerts`
- `pgepilot://profile`
- `pgepilot://collection-point/:code/:label?`

---

## Shared UI Surface With Web

`pge-app` and `pge-mobile` now share the same core user-facing surface for overlapping features:

| Shared function | Web | Mobile | Sync status |
|-----------------|-----|--------|-------------|
| Login | `Login.vue` | `LoginScreen.tsx` | Shared API, same JWT flow |
| Main overview | `Dashboard.vue` (`Přehled` in nav) | `DashboardScreen.tsx` (`Přehled`) | Aligned label and selected-domain behavior |
| Domains inventory | `Domains.vue` | `DomainsScreen.tsx` | Available on both platforms |
| Charts | `Grafy.vue` | `ChartsScreen.tsx` | Same API families, same Day/Week/Month/Year model |
| Installations inventory | `Instalace.vue` | `InstallationsScreen.tsx` | Available on both platforms |
| Alerts | `Alerty.vue` | `AlertsScreen.tsx` | Same role access: `admin`, `operator` |
| Collection point detail | `CPDetail.vue` | `CollectionPointDetailScreen.tsx` | Shared live/history/energy summary/command model |
| User context | `Nastaveni.vue` | `ProfileScreen.tsx` | Partial overlap via profile + selected domain |

Current intentional differences:

- web-only: `Uživatelé`
- mobile-only: `SmartBoxAdminPanel` in the profile area for admins

---

## Key Screens

### DashboardScreen

- domain selector using `/domains`
- selected domain persisted through `/users/me/preferences`
- KPI summary built from `/dashboard`
- collection point cards with direct navigation into detail

### CollectionPointDetailScreen

- live telemetry cards (PV, load, grid, battery, SoC)
- source selector
- day/week/month/year chart workflows
- energy summary
- command buttons for battery / export related actions
- admin-only SmartBox access helpers where relevant

### ChartsScreen

- selected domain + collection point context
- day/week/month/year range presets
- source selection (`effective` and backend-provided alternatives)
- power, energy and SoC visualization
- energy KPI cards for the selected period

### AlertsScreen

- alert feed backed by `/alerts`
- severity-oriented presentation
- collection point + domain context in each item

### ProfileScreen

- user identity and role
- selected domain summary
- logout
- SmartBox admin panel for admins

---

## API Integration

The mobile app uses the same API v2 as the web app. See [06-api-reference.md](06-api-reference.md) for the full endpoint list.

Current endpoints used by the mobile client:

- `POST /api/v2/auth/login`
- `GET /api/v2/domains`
- `GET /api/v2/dashboard`
- `GET /api/v2/summary`
- `GET /api/v2/users/me/preferences`
- `PUT /api/v2/users/me/preferences`
- `GET /api/v2/collection-points/:code`
- `GET /api/v2/collection-points/:code/battery-mode`
- `GET /api/v2/collection-points/:code/live`
- `GET /api/v2/collection-points/:code/history`
- `GET /api/v2/collection-points/:code/energy-summary`
- `POST /api/v2/commands`
- `GET /api/v2/alerts`

Auth model:

- JWT token stored in AsyncStorage
- token attached as `Authorization: Bearer <token>`
- local session cleared automatically on `401`

---

## Local Project Notes

Important local files:

- `src/navigation/RootNavigator.tsx`
- `src/services/api.ts`
- `src/screens/dashboard/DashboardScreen.tsx`
- `src/screens/dashboard/CollectionPointDetailScreen.tsx`
- `src/screens/charts/ChartsScreen.tsx`
- `src/screens/alerts/AlertsScreen.tsx`
- `src/screens/profile/ProfileScreen.tsx`

Release and field docs remain relevant:

- `docs/ios-testflight-checklist.md`
- `docs/android-release-checklist.md`
- `docs/smartbox-field-checklist.md`

---

## Known Gaps

- mobile documentation had been stale and marked as "planned"; that is no longer true
- web and mobile do not expose exactly the same information architecture outside the shared customer surface
- there is no shared package/component library yet; synchronization is currently maintained by convention and code review
