# 09 -- Development Roadmap

> Current production state, completed milestones, and prioritized work items.
> Last updated: 2026-04-12

---

## Production State (2026-04-12)

| Component | Status | Details |
|-----------|--------|---------|
| Legacy PgePilot | Production | 35 collection points (GoodWe, SolaX, SmartBox), health check, relay control |
| pge_control DB | Production | 28 tables (+ sems_plant_discovery), 35 CP, 26 devices, 40 machines, 39 connectors, realtime sync |
| pge_data DB | Production | 81 tables across raw/source history, derived/reporting datasets, energy lineage, and forecast + OTE spot import table in service DB |
| API v2 | Production | 36+ endpoints, JWT auth, forecast API, task API, usage-based history resolver |
| PGE App (web) | Production (synchronized) | Dashboard, CPDetail, Charts, Alerts, Domains, Users, Instalace, Nastaveni. Production source checkout is clean `main@3d7e6bb`; live bundle serves `/assets/index-lGjKNcFm.js`. |
| Forecast system | Production | PV (forecast.solar Pro), load (profiling), weather (Open-Meteo), adaptive correction |
| JobManager scheduler | Production | Current recurring runtime is DB-backed through `task_definitions`; legacy `pgepilot-js` scheduler block stays commented on current `main`. Runtime is clean `main@4346047`. |
| SmartBox SB1 | Production (monitoring) | 4 microservices, Modbus polling, RPC working, 2139 rpc_kv records. Plant VA_SB (ID 202) registered as pull connector. Live code: `devva@be59807`; `origin/devva` is one additive commit ahead at `5322f3f` (Deye driver merge), not yet deployed on SB1. |
| SmartBox sb4 | Dev (at home) | Same codebase as SB1 (`devva@5322f3f`), rsync deployed 2026-04-11. GoodWe inverter disabled (shared LAN with sb1). Deye driver smoke-tested OK. |
| SmartBox TOU | In progress | Spec ready, not yet implemented on real SB |
| sb-manager | Production (historically unstable) | 161 total restarts recorded by PM2; currently online with ~38h uptime (verified 2026-04-12) |
| Email API | Production | 5 profiles (servis, automat, technika, obchod, info) |
| Connector self-governance | Production | Budget trait, cache, enabled flags |
| Worker runtime sync | **DONE** | worker1/2/3 were backed up and reset on 2026-04-10; refreshed to clean `main@a04e0ba` on 2026-04-12 and later unified with `pgepilot_service` on `main@a30c78c` during GoodWe task 18 stabilization |
| GoodWe task 18 stabilization | **DONE / monitor** | Hard `HTTP 500` root cause was a general GoodWe historical backfill problem: one 7-day `GetStationHistoryDataChart` payload could exhaust PHP memory, and reverse backfill retried the same empty older chunk forever. First mitigation (`131080f`) reduced requested targets to the canonical minimum. Final fix (`a30c78c`) changed GoodWe historical ingest to per-day pulls and stores `plant_config.goodwe_reverse_empty_before_date` when the chunk immediately before current `min_date` is empty. Verified on production 2026-04-12: `BD41` 7-day reverse backfill completed `OK` with 10045 rows saved; `FVE_DF55` now returns `reverse_exhausted` on the second run instead of repeating the same empty window; `task_definition_id=18` had zero failed rows after the deploy window starting 17:40. |
| VPS tunnel keepalive | **DONE** | Appended `ClientAliveInterval 30` + `ClientAliveCountMax 3` to `/etc/ssh/sshd_config` on VPS ra-energity.cz. Before: rebooted SB devices left zombie tunnel sessions on VPS (stale sshd processes for `ra_tunnel` user) blocking port re-binding for hours. After: VPS detects dead tunnels within 90s and cleans them. Also cleaned 5 accumulated zombie sessions (from Apr 01 to Apr 11) with `sudo pkill -u ra_tunnel`. Backup: `/etc/ssh/sshd_config.bak-20260412`. (2026-04-12) |
| HW watchdog sb1+sb4+sb7 | **DONE** | Enabled `RuntimeWatchdogSec=30` in `/etc/systemd/system.conf` on sb1 (BCM2835 watchdog, hw timeout 1min), sb4 (STM32 IWDG, 30s), sb7 (STM32 IWDG, 30s). Applied via `sed -i` + `systemctl daemon-reexec`. Triggered by real incident: sb1 froze during GoodWe Modbus TCP collision between sb1 and sb4 (both polling same inverter) and required physical power-cycle. Also changed `StartLimitIntervalSec=0` on ra-tunnel.service on all 3 devices (was `300/10` — caused tunnel to give up after 10 restarts). Added `enable_hardware_watchdog()` to `sb_bootstrap.sh` template for future devices. Added `getSystemInfo()` to sb-ops-agent `/health` response (reports watchdog state, uptime, kernel, disk free). Added compliance check in sb-manager `deviceHealthMonitor.js` — logs `WATCHDOG_OFF` warning if device has watchdog disabled. Backups on each device: `/etc/systemd/system.conf.bak-20260412`. (2026-04-12) |
| Deye driver merge | **DONE** | `origin/dev_deye` was 10 commits behind `devva` (missing security bind, port layout, mobile redesign, machine_id, DC poll optimization). Full merge would break production. Analyzed all 19 differing files — only 6 were Deye-related. Surgical cherry-pick: 4 new files (`solarman_driver.py` 303 lines Solarman V5 protocol, `deye_inverter.py` 410 lines DeyeInverterDriver, `commands_deye.yaml` 13 commands, `functions_deye_poll.yaml` 4 poll categories), 1 replaced (`modbus_reg_deye.yaml` 5-line stub → 353-line SUN-xK-SG04LP3-EU register map, 36 registers), 2 lines added to `device_manager.py` (dict entry + comment). Skipped: poll loop refactor (23 lines print spam), rpc_client test config. Created `devva_deye` branch, merged FF into `devva`, pushed. Commit `5322f3f`. Smoke-tested on sb4: imports OK, 36 registers parsed, 13 commands loaded, Modbus connect failure to TEST-NET-1 graceful. `origin/dev_deye` and `origin/copilot/add-support-for-deye-inverter` (bigger Copilot variant, 907-line driver, user chose smaller) left untouched. (2026-04-11) |
| sb4 deploy | **DONE** | sb4 is M5Stack CoreMP135 at home, shared LAN with sb1. `/opt/energity/sb/` is NOT a git repo — deployed via rsync from Mac. Procedure: tar backup (`/tmp/sb-backup-20260411-212829.tar.gz`), stop 7 services, rsync with excludes (devices_config.yaml, rpc_client_config.yaml, smartbox_config.yaml, local_database/*.db, .venv/), chown energity:energity, start services. Verified: 7/7 active running, sb-ops-agent allUp=true 6/6, DeyeInverterDriver registered in runtime, GoodWe Connected to 192.168.0.70:502 (when enabled), sensor_data.db growing. Later updated to `devva@e5be146` (JSON cache + smartboxSendData fix). (2026-04-11, updated 2026-04-12) |
| sb7 full setup | **DONE** | M5Stack CoreMP135, n=7, base port 20060, SSH port 20062. Provisioned from scratch: hostname `sb7`, watchdog `RuntimeWatchdogSec=30`, `StartLimitIntervalSec=0` on ra-tunnel, root SSH key added (PubkeyAuthentication was `no`, fixed), rsync `devva@e5be146` code, python3-venv + deps (flask, fastapi, uvicorn, pymodbus, pyyaml, requests), 7 systemd service units (sensor-db, logs-db, device-controller, comm-controller, rpc-client, config-api, web-interface) — installed manually because bundled `install_systemd_services.sh` had wrong paths from SB1 (`/home/energity1/...`). devices_config.yaml set for Deye SUN-xK-SG04LP3-EU via SolarmanDriver (LSW-3 dongle, port 8899, enabled:false — activate on site). sb-manager label `sb7`, notes `Chlumcany Skola`. (2026-04-12) |
| YAML JSON cache | **DONE** | PyYAML parsing of `modbus_reg_goodwe.yaml` (2103 registers, 2113 lines) takes 37 seconds on ARM Cortex-A7 (M5Stack CoreMP135 500 MHz). Added transparent JSON cache in `_load_yaml_cached()` static method — compares mtime of `.yaml` vs `.cache.json`, rebuilds cache only when YAML changes. JSON load: 162ms (227x faster). Applied to all 3 YAML load paths: `_load_registers_config()` in `modbus_device.py`, `_load_poll_functions_from_yaml()` and `_load_commands_from_yaml()` in `base_device.py`. Device-controller startup with cache: 4s (was 55s). Benchmarked on sb4 with real GoodWe config. Commit `0a0f747`. (2026-04-12) |
| smartboxSendData fix | **DONE** | Fixed 6 bugs in `_get_current_telemetry()` field mapping in `smartbox_service.py`: (1) grid_power was always 0 — mapping used `meter_power` but GoodWe register is `meter_total_active_power`; (2) load_power mapped to `total_backup_load_power` which doesn't exist, fixed to `total_load_power`; (3) PV power not summed — added `pv1_power + pv2_power` fallback when `pv_total_power` unavailable; (4) battery power only read battery1 — now sums `battery1_power + battery2_power`; (5) grid sign normalization — GoodWe raw +import/-export, cloud wants +export/-import, applied flip + uint16 rollover fix (>32767 → negative); (6) added `clientTs` to top-level payload. Battery sign OK (GoodWe raw +discharge/-charge matches cloud contract). Verified on sb4 against real GoodWe: `smartboxSendData` HTTP 200 OK to `service.pgepilot.cz/rpc`. Energy windows still send daily totals (see H17). Commit `e5be146`. (2026-04-12) |
| Sign conventions doc | **DONE** | Created `15-sign-conventions.md`. Standard convention: positive = charging/import, negative = discharging/export (from house perspective). Documented per-manufacturer differences: GoodWe battery raw +discharge/-charge (cloud OK, dashboard needs flip), GoodWe grid raw +import/-export (needs flip for cloud +export/-import), Deye native matches our convention. Verified GoodWe empirically on sb4: `battery1_power=-1750W` while `SOC 49→50%` = negative means charging. Includes per-register addresses, implementation code snippets, and task H16 for driver-level normalization. (2026-04-12) |
| LCD dashboard | **DONE** | M5Stack CoreMP135 has built-in 2" IPS LCD (ILI9342C, 320x240, RGB565, SPI 50MHz, `/dev/fb1`) and capacitive touch (EP0110M09, `/dev/input/event0`). Built Python dashboard (`display_dashboard.py`) using Pillow (DejaVu fonts) + evdev for touch. 3 swipeable pages: (1) Status — services 7/7, IP, tunnel, watchdog, inverter type, uptime; (2) Energy Flow — 4-box layout: FOTOVOLTAIKA, SPOTREBA, NABIJENI/VYBIJENI, PRODEJ/NAKUP with live values in kW, SOC bar, battery sign indicator; (3) Data table — per-value cards with icons. Auto-refresh every 15s, reads from sensor-db `/get_latest_data` API. Touch swipe detection via `evdev.InputDevice` in background thread (BTN_TOUCH + ABS_X delta > 30px). Alias from `/opt/energity/sb-ops-agent/device_alias.txt`. Dark background for contrast. Czech labels. Installed as systemd service `energity-display.service` (Restart=always). GoodWe battery sign verified empirically: raw negative = charging (SOC rising), applied correct label in dashboard. (2026-04-12) |

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
| JobManager scheduler | 03-29 to 04-07 | Runtime now verified as DB-backed `task_definitions` + workers; legacy scheduler endpoints remain commented in current `pgepilot-js` |
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
| Task 18 rewritten | 04-04 | GetStationHistoryDataChart (1min, 57 registers) replaces GetPlantPowerChart (5min, 6 curves). Initial output: power_1m + power_bf |
| GetChartByPlant | 04-04 | Replaces GetPlantPowerChart in getPlantPowerChartAll(). 12 curves (+ per-phase meter/load + SOC2) |
| power_1m tables created | 04-04 | tpl_power_1m + 17 GoodWe plant tables (57 register columns + 5 computed) |
| sems_plant_discovery | 04-04 | 228 SEMS plants cataloged in pge_control, in_pgepilot flag |
| battery_soc2_pct column | 04-04 | Added to all power_bf tables |
| Discovery scripts | 04-04 | goodwe_discovery.php (metadata + backfill_start_date), sems_fetch_all_plants.php (228 plants) |
| GoodweSemsWeb new methods | 04-04 | getPowerStationList, getPlantMonitor, getInventers, getMonitorDetail, getStationHistoryData |
| GoodweSemsWeb credentials | 04-04 | Old account [REDACTED] deactivated (100029), new account [REDACTED] working |
| OTE PT15M importer | 04-07 | `GET /ote/import` added in service, stores into `ote_day_ahead_prices` |
| OTE tasks 30/31 | 04-07 | JobManager-verified imports for today (`00:05`) and today+tomorrow (`12:10`) |
| Git audit refresh | 04-09 to 04-10 | `pgepilot-service` `main` at `b578bd8`, `pge-app` `main` at `05e61e0`; production workers, jobmanager cleanup, and live `pge-app` checkout were resynchronized. Auth runtime provenance remains unresolved. |
| Runtime resync cleanup | 04-10 | worker1/2/3 reset to `main@b578bd8`, `pge-app` reset + rebuilt to `main@05e61e0`, jobmanager untracked backups removed during cleanup. Worker pre-sync backups live under `/root/runtime-sync-backups/2026-04-10` |
| History usage resolver + energy lineage | 04-12 | API detail now exposes `history_usage_options`; `/history` + `/energy-summary` accept `usage`; energy aggregation prefers `power_1m`, falls back to `power_rt`, and stores source lineage metadata |
| History pipeline cutover | 04-12 | GoodWe backfill writes canonical `pge_data.{code}_power_1m`, SmartBox `smartboxSendData` writes canonical `power_1m` + reported `energy_15m`, history policy points to `power_1m`, and `power_bf` is now primarily a reporting profile resolved over canonical history |
| Production deploy of history cutover | 04-12 | `pgepilot_service` + workers 1/2/3 synced to `a04e0ba`, migration `009_history_lineage_and_policy_cutover.sql` applied, `pge-app` synced to `3d7e6bb`, rebuilt, and PM2-restarted |
| Service-only auth/helper follow-up | 04-12 | `pgepilot_service` briefly advanced to `da784a6` (`fix: bind smartbox local auth middleware helper`) while workers remained on `a04e0ba`. This temporary runtime split was later closed by the task 18 stabilization deploy to `a30c78c`. |

---

## Unfinished / Handoff Items

Items left incomplete from the last development session. Start here before new work.

| # | Item | Status | Detail |
|---|------|--------|--------|
| H1 | ~~Synchronize worker1/2/3 with git~~ | **DONE** | Backups stored under `/root/runtime-sync-backups/2026-04-10`, then workers were refreshed again to clean `main@a04e0ba` on 2026-04-12. |
| H2 | ~~Synchronize live PGE App checkout with git~~ | **DONE** | `pgepilot_servicedesk:/home/app2/pge-app` is now clean `main@3d7e6bb` and rebuilt in place. |
| H3 | **Prove auth runtime provenance** | UNKNOWN | `pgepilot_auth_srv` runs from `/app` with no visible git checkout. Need to link it to a repo/image source of truth. |
| H4 | ~~Create tpl_power_bf table~~ | **DONE** | tpl_power_1m created (57 register + 5 computed columns). 17 GoodWe plant tables created. Task 18 rewritten to use power_1m. |
| H5 | **Activate task_definitions 20-22** | PARTIAL | `22 recordRealtimeToHistory` was activated on 2026-04-12. `20` and `21` remain disabled until their legacy sync behavior is either removed or explicitly re-approved. |
| H6 | **SB1 poll warning** | LOW | Communication controller logs poll as "failed" (ok=None). Either server adds `ok: true` to smartboxPoll response, or fix comm controller parsing. |
| H7 | **SB1 NTP warning** | LOW | "NTP not initialized, using system time" -- not critical but should be resolved. |
| H8 | ~~Git push SB1 changes~~ | **DONE** | `Holbytlo/sb` `devva` currently points to commit `5322f3f` (`feat: add Deye inverter driver`) |
| H9 | **SolaX backfill re-enable** | WAITING | Set connector_config backfill_enabled=1 for SOLAX_CLOUD after budget logic is proven reliable. |
| ~~H10~~ | ~~GoodweSems OpenAPI token refresh broken~~ | **DONE** (2026-04-04) | Fixed: in-memory token cache + auto-retry on 100002. Root cause: INI file race condition between workers + no token refresh on expired response. Commit `248351d`. |
| H11 | **Watchdog in provisioning** | TODO | `RuntimeWatchdogSec=30` must be enabled on every new SB device during provisioning. Currently done manually post-install. Should be added to `sb_bootstrap.sh` (Debian variant) or baked into base image. Without watchdog, a frozen SB requires physical power-cycle. |
| H12 | **GoodWe scale factor N/A** | LOW | `modbus_reg_goodwe.yaml` has `scale: N/A` on ~15 registers. Causes `Invalid scale factor` warnings every poll cycle on both sb1 and sb4. Fix: replace `N/A` with numeric scale or remove those registers from poll config. Also causes `Serial=None, Model=None` at device init. |
| H13 | **sb7 Deye deploy** | NEXT | sb7 goes to customer with Deye inverter (SUN-xK-SG04LP3-EU). Needs: get sb7 online, rsync `devva`, configure `devices_config.yaml` with real Deye IP/unit_id, test against real HW. Solarman V5 transport may be needed if customer has LSW-3 dongle instead of direct Modbus TCP. |
| H14 | **sb1 deploy of devva@5322f3f** | OPTIONAL | sb1 still runs `be59807`; rsync deploy of the newer additive Deye commit has not been done yet. GoodWe runtime is currently stable. Deploy when convenient -- same rsync procedure as sb4. |
| H15 | **Stale tunnel cleanup on VPS** | **DONE** | Fixed with `ClientAliveInterval 30` + `ClientAliveCountMax 3` (2026-04-12). Old zombie sessions accumulating since Apr 01 were cleaned with `sudo pkill -u ra_tunnel`. |
| H16 | **Sign normalization in drivers** | TODO | Drivers should normalize battery/grid signs so all consumers (dashboard, cloud, web) get consistent convention. Currently each consumer flips signs independently. See `15-sign-conventions.md`. |
| H17 | **energyWindows real calculation** | TODO | `smartboxSendData` sends daily kWh totals as 5/10/15/60m windows — wrong. Need SB-side ring buffer to compute actual rolling energy per window. Cloud backend expects real windowed values. |
| H21 | **GoodWe task 18 HTTP 500** | **DONE / monitor** | Root cause was general GoodWe backfill logic, not one broken plant. `GetStationHistoryDataChart` for a 7-day range could OOM PHP; empty reverse windows were retried forever. Fixed by `131080f` (smaller target set) + `a30c78c` (day-by-day ingest + reverse stop marker in `plant_config`). Production canaries on `BD41` and `FVE_DF55` passed on 2026-04-12; keep monitoring for a few scheduler cycles. |
| H18 | **sb-manager alias field** | TODO | PATCH API rejects `alias` field (unrecognized key). Column exists in DB but not in zod schema. Add to PATCH validation + propagate to device via health poll response. |
| H19 | **sb4 WiFi stability** | LOW | USB WiFi dongle (Ralink RT5370) drops connection repeatedly. Ethernet preferred for stable operation. RTL8188GU dongle not supported (no kernel driver). |
| H20 | **GoodWe Modbus single-client** | KNOWN | Two SmartBoxes on same GoodWe inverter cause `Connection reset by peer` loops. Only one SB can poll per inverter. Documented in `02-smartbox-sbc.md`. |

---

## Priority 1 -- Complete (1-2 weeks)

| # | Task | Detail | Effort |
|---|------|--------|--------|
| 1.1 | ~~Installation page~~ | **DONE** on current `pge-app` `main` (`3d7e6bb`) | ~~Medium~~ |
| 1.2 | ~~Settings page~~ | **DONE** on current `pge-app` `main` (`3d7e6bb`) | ~~Medium~~ |
| 1.3 | Password change | Missing endpoint + UI | Small |
| 1.4 | ~~Sync crons to JobManager~~ | **DONE** -- migrated to task_definitions 20-22; `22` is now active, `20/21` remain intentionally disabled | ~~Medium~~ |
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
| 2.4 | SPOT optimization | OTE PT15M import is in production; next step is to use these prices for charge/discharge planning | Medium |
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
| 3.10 | sb-manager stability | Diagnose 156 historical restarts and keep uptime stable | Medium |
| 3.11 | Cron monitoring | Alert when sync fails | Medium |
| 3.12 | plant_config | Fill in for all plants (many missing) | Small |
| 3.13 | **Watchdog in provisioning** | Add `RuntimeWatchdogSec=30` to `sb_bootstrap.sh` or base image. Every new SB must have HW watchdog enabled at install time. Currently manual post-install step (see H11). | Small |
| 3.14 | **GoodWe scale N/A fix** | Fix `modbus_reg_goodwe.yaml` scale: N/A entries (see H12). Currently ~15 registers produce warnings every poll cycle. | Small |
| 3.15 | **sb1 code deploy** | Rsync `devva@e5be146` to sb1 (see H14). Live SB1 is still on `be59807`; deploy is additive and expected downtime is ~30s. Now includes JSON cache + smartboxSendData fix. | Small |
| 3.16 | **Sign normalization** | Normalize battery/grid signs in drivers, not in consumers. See H16 + `15-sign-conventions.md`. | Medium |
| 3.17 | **energyWindows real calculation** | SB-side rolling energy aggregation for 5/10/15/60m windows. See H17. | Medium |
| 3.18 | **sb-manager alias API** | Add `alias` to PATCH schema + propagate to device. See H18. | Small |

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
