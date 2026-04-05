# 09 -- Development Roadmap

> Current production state, completed milestones, and prioritized work items.
> Last updated: 2026-04-05

---

## Production State (2026-04-04)

| Component | Status | Details |
|-----------|--------|---------|
| Legacy PgePilot | Production | 35 collection points (GoodWe, SolaX, SmartBox), health check, relay control |
| pge_control DB | Production | 28 tables (+ sems_plant_discovery), 35 CP, 26 devices, 40 machines, 39 connectors, realtime sync |
| pge_data DB | Production | 81 tables (power_1m, energy_15m, forecast), sync crons + scheduler |
| API v2 | Production | 36+ endpoints, JWT auth, forecast API, task API |
| PGE App (web) | Production | Dashboard, CPDetail, Charts (aggregation), Alerts, Domains, Users, Instalace, Nastaveni |
| Forecast system | Production | PV (forecast.solar Pro), load (profiling), weather (Open-Meteo), adaptive correction |
| JobManager scheduler | Needs verification | Older docs describe 3 forecast tasks, but current `pgepilot-js` `main` has the scheduler block commented out |
| SmartBox SB1 | Production (monitoring) | 4 microservices, Modbus polling, RPC working, 2139 rpc_kv records. Plant VA_SB (ID 202) registered as pull connector. |
| SmartBox TOU | In progress | Spec ready, not yet implemented on real SB |
| sb-manager | Production (unstable) | 155 restarts in 4 days |
| Email API | Production | 5 profiles (servis, automat, technika, obchod, info) |
| Connector self-governance | Production | Budget trait, cache, enabled flags |

---

## Recently Completed (2026-03-28 -- 2026-04-04)

| Item | Date | Detail |
|------|------|--------|
| pgepilot_data migration | 03-28 | Moved from pgedata.cz to local pgepilot.cz |
| SolaX SoC fix | 03-28 | battery_capacity set for all machines |
| VA 2-year backfill | 03-28 | Historical data 2024-03 to 2026-02 |
| PV forecast | 03-29 | forecast.solar Professional, multi-string, 6 days, 15min |
| Load forecast | 03-29 | Weekly profile, holidays, seasonal/temperature correction |
| Adaptive correction | 03-29 | EMA alpha=0.15, daily comparison |
| Forecast UI | 03-29 | Historical note: older docs referenced `/predikce`, but current `pge-app` `main` has no dedicated forecast route/component |
| Chart aggregation | 03-29 | Year: months/weeks, Month: days/weeks, Week: days/hours |
| JobManager scheduler | 03-29 | Historical note: current `pgepilot-js` `main` does not expose active scheduler endpoints in git |
| Email profiles | 03-29 | 5 profiles configured |
| Collect task (29) enabled | 04-04 | GoodWe + SolaX data flowing again |
| SolaX API fixed | 04-04 | Reset at midnight, monitor email sent |
| SolaX HC enabled | 04-04 | 15 plants, reads from cache |
| HC quiet hours fix | 04-04 | Night window 18:00-07:00, severity=2 for non-fault |
| compute_energy_15m (task 23) | 04-04 | Fixed column names, tested (7721 buckets), enabled |
| SmartBox SB1 fixed | 04-04 | Config pointed to production, comm controller running |
| Connector self-governance | 04-04 | ConnectorBudgetTrait, cache, enabled flags per connector |
| Docker-compose fix | 04-04 | servicedesk added to nginx-proxy-manager network |
| api_usage retention | 04-04 | Cron deletes rows older than 30 days |
| VA_SB plant registered | 04-04 | SmartBox pull connector, plant ID 202, tables va_sb_power_rt + va_sb_energy_15m |
| Task 18 rewritten | 04-04 | GetStationHistoryDataChart (1min, 57 registers) replaces GetPlantPowerChart (5min, 6 curves). Output: power_1m + power_bf |
| GetChartByPlant | 04-04 | Replaces GetPlantPowerChart in getPlantPowerChartAll(). 12 curves (+ per-phase meter/load + SOC2) |
| power_1m tables created | 04-04 | tpl_power_1m + 17 GoodWe plant tables (57 register columns + 5 computed) |
| sems_plant_discovery | 04-04 | 228 SEMS plants cataloged in pge_control, in_pgepilot flag |
| battery_soc2_pct column | 04-04 | Added to all power_bf tables |
| Discovery scripts | 04-04 | goodwe_discovery.php (metadata + backfill_start_date), sems_fetch_all_plants.php (228 plants) |
| GoodweSemsWeb new methods | 04-04 | getPowerStationList, getPlantMonitor, getInventers, getMonitorDetail, getStationHistoryData |
| GoodweSemsWeb credentials | 04-04 | Old account [REDACTED] deactivated (100029), new account [REDACTED] working |
| Git push | 04-05 audit | Current git HEADs verified: `pgepilot-service` `248351d`, `pge-app` `aaeff13`, `pgepilot-js` `4346047`, `sb` `be59807` |

---

## Unfinished / Handoff Items (from 2026-04-04 session)

Items left incomplete from the last development session. Start here before new work.

| # | Item | Status | Detail |
|---|------|--------|--------|
| H1 | **Deploy GoodweSems.php cache** | NOT DEPLOYED | 4 new cache methods (GetDeviceAttribute, getInverterEnergyUsedDetail, GetInverterPowerV1, GetPlantPower) exist in /tmp/GoodweSems.php on local Mac. Need docker cp to all 4 containers (service + worker1/2/3), both src/ and wsrc/ paths. Then git commit + push. |
| H2 | **Deploy SolaxCloud.php getMpptInfo cache** | NOT DEPLOYED | Local file /tmp/SolaxCloud.php. Same deploy procedure. |
| H3 | **GoodweSemsWeb.php needs ConnectorBudgetTrait** | NOT STARTED | Separate class for web scraping GoodWe history. Has 4 read methods without cache or budget checks. |
| H4 | ~~Create tpl_power_bf table~~ | **DONE** | tpl_power_1m created (57 register + 5 computed columns). 17 GoodWe plant tables created. Task 18 rewritten to use power_1m. |
| H5 | **Activate task_definitions 20-22** | READY | Sync tasks migrated from crontab but currently disabled. Need to verify they work, then enable. |
| H6 | **SB1 poll warning** | LOW | Communication controller logs poll as "failed" (ok=None). Either server adds `ok: true` to smartboxPoll response, or fix comm controller parsing. |
| H7 | **SB1 NTP warning** | LOW | "NTP not initialized, using system time" -- not critical but should be resolved. |
| H8 | ~~Git push SB1 changes~~ | **DONE** | `Holbytlo/sb` `devva` currently points to commit `be59807` (`config: switch SB1 to production`) |
| H9 | **SolaX backfill re-enable** | WAITING | Set connector_config backfill_enabled=1 for SOLAX_CLOUD after budget logic is proven reliable. |
| ~~H10~~ | ~~GoodweSems OpenAPI token refresh broken~~ | **DONE** (2026-04-04) | Fixed: in-memory token cache + auto-retry on 100002. Root cause: INI file race condition between workers + no token refresh on expired response. Commit `248351d`. |

---

## Priority 1 -- Complete (1-2 weeks)

| # | Task | Detail | Effort |
|---|------|--------|--------|
| 1.1 | ~~Installation page~~ | **DONE** on current `pge-app` `main` (`aaeff13`) | ~~Medium~~ |
| 1.2 | ~~Settings page~~ | **DONE** on current `pge-app` `main` (`aaeff13`) | ~~Medium~~ |
| 1.3 | Password change | Missing endpoint + UI | Small |
| 1.4 | ~~Sync crons to JobManager~~ | **DONE** -- migrated to task_definitions 20-22 (currently disabled, need activation) | ~~Medium~~ |
| 1.5 | PV forecast for all CPs | Add pv_forecast_config for more plants (currently only VA + Kder Veltrusy) | Medium |
| 1.6 | SolaX backfill | Nov 2025 - Mar 2026 not completed (was killed) | Medium |
| 1.7 | GoodWe backfill | Complete history for all GW plants | Medium |

---

## Priority 2 -- New Features (2-4 weeks)

| # | Task | Detail | Effort |
|---|------|--------|--------|
| 2.1 | Command execution | cp_commands -> connector poll -> device. Backend exists, needs execution | Large |
| 2.2 | SB TOU engine | sb.* API on real SB1 (spec: zadani_api_commandu_sb_plant_v0_1.docx) | Large |
| 2.3 | Mobile app | React Native, shares API v2 (spec: Tvorba mob app/) | Large |
| 2.4 | SPOT optimization | Use OTE prices for charge/discharge planning | Medium |
| 2.5 | BOILER/EVSE/POOL in TOU | Extend TOU engine to more device types | Medium |
| 2.6 | Forecast in Charts | Overlay forecast on historical charts; current `main` has no dedicated `/predikce` page | Medium |
| 2.7 | Docker compose | Restructure Docker infrastructure | Medium |

---

## Priority 3 -- Hardening (ongoing)

| # | Task | Detail | Effort |
|---|------|--------|--------|
| 3.1 | DB password | Change from hardcoded to env var | Small |
| 3.2 | JWT secrets | Change defaults to random | Small |
| 3.3 | SMTP password | Move from code to env var | Small |
| 3.4 | db.pgepilot.cz | Block external access on port 3306 | Small |
| 3.5 | Backup expansion | Add pge_control, pge_data, pgep_tasks to backup | Small |
| 3.6 | Tailwind CDN | npm install tailwindcss | Small |
| 3.7 | Auth middleware | Enable on all write endpoints | Medium |
| 3.8 | Dead proxy entries | Remove sicak, calc, nab, taskmanager from NPM | Small |
| 3.9 | SB1 failing services | Stop sb-test-3000/3001 | Small |
| 3.10 | sb-manager stability | Diagnose 155 restarts in 4 days | Medium |
| 3.11 | Cron monitoring | Alert when sync fails | Medium |
| 3.12 | plant_config | Fill in for all plants (many missing) | Small |

---

## Priority 4 -- Future

| # | Task | Detail |
|---|------|--------|
| 4.1 | Energy communities | EANd/EANo sharing, export_alloc_15m |
| 4.2 | SB plugin framework | sb-core + drivers as plugins, OTA updater |
| 4.3 | Canary rollout | Gradual update rollout to SB devices |
| 4.4 | Virtual battery | Battery aggregation across CPs |
| 4.5 | HVAC / heat pumps | Additional device type |

---

## Recommended Work Order

```
Week 1:  1.4 (Sync to scheduler) + 1.5 (PV forecast for more CPs) + 3.1-3.4 (security)
Week 2:  1.1 (Installation page) + 1.2 (Settings) + 1.3 (password change) + 3.5 (backup)
Week 3:  1.6 (SolaX backfill) + 1.7 (GW backfill) + 2.7 (Docker compose)
Week 4+: 2.1 (Command execution) + 2.2 (SB TOU engine)
```

---

## Active Plants

### GoodWe (17 total, 15 with health check)

| Code | Storage | Inverter SN | Notes |
|------|---------|-------------|-------|
| BD41 | bd41 | 9010KETU227W2085 | |
| BETO1 | beto1 | (missing SN) | Energy backfill skipped |
| BPB1 | bpb1 | 9010KETU219W0731 | |
| CRZ1 | crz1 | 9010KETU21CW3722 | |
| FVE_Bol | fve_bol | 9015KEUB251L0044 | |
| FVE_CERN | fve_cern | 5025KETT254S0053 | |
| FVE_DF55 | fve_df55 | 5010KETU231W0996 | |
| FVE_TRC | fve_trc | 5010KETU232W9097 | |
| FVE_ZM1 | fve_zm1 | 5010KETU22BW2647 | |
| KCE1 | kce1 | 9010KETU227W2082 | |
| Kost_ou | kost_ou | 9020KETT235W0807 | |
| LSS1 | lss1 | 5010KETU232W6822 | |
| Mik_Chl | mik_chl | 5010KETU22BW2609 | |
| RYB_VIN | ryb_vin | 9010KETU22BW1789 | |
| RYBCap | rybcap | 9010KETU224W4101 | |
| SVP1 | svp1 | (missing SN) | Energy backfill skipped |
| VA | va | 9025KETT244L0024 | 2-year backfill done |

### SolaX (15 plants)

All SolaX plants have health check enabled, reading from cache (0 extra API calls). Backfill disabled for SolaX (API limit protection).

### SmartBox (2 collection points)

- SBX_DEYE25 -- SB1 at Rozdrojovice, GoodWe ET+ inverter via Modbus TCP
- VA_SB -- SmartBox pull connector (plant ID 202), health_url: https://sb1-health.ra-energity.cz/telemetry
