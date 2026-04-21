# 12 -- Forecasting System

> PV production forecast, load forecast, weather data, adaptive correction, and Open-Meteo physical PV model.
> Last updated: 2026-04-21 (audit against production reality)

---

## Overview

PgePilot runs three forecast-related pipelines plus an adaptive correction layer. Current production recurring execution is DB-backed through `pgep_tasks.task_definitions`, not through the older `pgepilot-js` `/scheduled_tasks` block (which is commented out in current main). The Open-Meteo physical PV model (task `26`) and adaptive correction (task `27`) are the only forecast pipelines actively running; `forecast.solar` (task `24`) and the load forecast (task `25`) have been disabled since 2026-04-04.

| Pipeline | Source | Granularity | Horizon | Schedule | Status |
|----------|--------|-------------|---------|----------|--------|
| PV forecast -- forecast.solar | forecast.solar Professional | 15 min | 6 days | DB def `24`, `every:3600seconds` | **Disabled** since 2026-04-04; scripts retained |
| Load forecast | Internal profiling | 15 min | 7 days | DB def `25`, `every:3600seconds` | **Disabled** since 2026-04-04; scripts retained |
| PV forecast -- Open-Meteo physical model | Open-Meteo (DWD ICON 15min + Forecast API) | 15 min (0-48h) / 1h (3-16d) | ~16 days | `every:3600seconds` via task `26` | **Production** (`pv_forecast_om.php`) |
| Weather forecast | Open-Meteo API | hourly (+15min DWD subset) | ~16 days | Fetched as part of task `26` | **Production** |
| Adaptive correction | EMA (alpha=0.15) | Daily | -- | `daily:01:05` via task `27` | **Production** (`forecast_correction.php`) |

---

## PV Forecast (forecast.solar)

### How It Works

Uses the [forecast.solar](https://forecast.solar) Professional API to predict solar production based on panel configuration and weather models.

| Property | Value |
|----------|-------|
| API tier | Professional |
| API key | [REDACTED] |
| Granularity | 15-minute intervals |
| Horizon | 6 days ahead |
| Multi-string | Yes (separate azimuth/tilt per array) |
| DB task | `24` (`pgep_tasks.task_definitions`) |
| Schedule | `every:3600seconds`, **currently disabled** (last run 2026-04-04) |
| Manual trigger | `POST /api/v2/tasks/run-pv-forecast` |
| Storage | `pge_data.pv_forecast` (`source='forecast_solar'`) |
| Config table | `pge_control.pv_forecast_config` |
| Script | `pgepilot-service/scripts/pv_forecast.php` |

### Currently Enabled Plants

5 of 47 active collection points have PV forecast enabled (checked 2026-04-21):

| CP code | Label | kWp | tier | forecast_source | model_version | Note |
|---------|-------|-----|------|-----------------|---------------|------|
| `va` | VA | 11.7 | basic | forecast_solar | v2_factored | Real plant; OM fallback when task 24 disabled |
| `kder_vel` | Kder Veltrusy | 10.0 | basic | forecast_solar | v1_simple | Real plant; OM fallback when task 24 disabled |
| `demo_kd` | Demo Kulturní dům | 18.0 | adaptive | manual (synth) | v2_factored | Synthetic demo profile |
| `demo_rad` | Demo Radnice | 26.0 | adaptive | manual (synth) | v2_factored | Synthetic demo profile |
| `demo_zs` | Demo Základní škola | 42.0 | adaptive | manual (synth) | v2_factored | Synthetic demo profile |

Of the 5 rows, only **2 cover real plants** (VA, Kder Veltrusy); the other 3 are synthetic demo data for UI testing, written with `source='manual'`. Historical `source='forecast_solar'` rows exist but have not been updated since task 24 was disabled on 2026-04-04 — in practice these two real plants are served by task 26 (OpenMeteo physical model).

**Expanding to more plants** is Priority 1.5 in the roadmap -- requires adding rows to `pv_forecast_config` with panel parameters (azimuth, tilt, capacity per string).

### pv_forecast_config Table (`pge_control`)

```sql
-- Full schema as of 2026-04-21:
id                    CHAR(36)                                             -- FK to cp_collection_points.id
enabled               TINYINT(1)      DEFAULT 0
tier                  ENUM('free','basic','adaptive') DEFAULT 'basic'      -- business segmentation
pv_kwp                DECIMAL(6,2)
strings_json          LONGTEXT                                             -- [{"kwp":..,"declination":..,"azimuth":..}, ...]
declination_deg       SMALLINT        DEFAULT 35                           -- fallback panel tilt
azimuth_deg           SMALLINT        DEFAULT 0                            -- fallback panel azimuth (0=north, 180=south)
latitude              DECIMAL(10,7)
longitude             DECIMAL(10,7)
forecast_source       ENUM('forecast_solar','solcast','openmeteo_ghi','manual') DEFAULT 'forecast_solar'
correction_factor     DECIMAL(4,2)    DEFAULT 1.00                         -- EMA factor for forecast_solar branch
correction_factor_om  DECIMAL(4,2)    DEFAULT 1.00                         -- separate EMA factor for openmeteo branch
notes                 VARCHAR(500)
updated_at            DATETIME
model_version         ENUM('v1_simple','v2_factored') DEFAULT 'v1_simple'
shade_enabled         TINYINT(1)      DEFAULT 0                            -- factor-model horizon shading
snow_model            TINYINT(1)      DEFAULT 0                            -- factor-model snow cover attenuation
```

`correction_factor` and `correction_factor_om` are maintained separately per source because the two pipelines produce independently biased predictions. The `v2_factored` model in `pv_forecast_om.php` additionally applies per-hour multiplicative factors for shade horizon, snow cover, cloud nonlinearity, and azimuthal drift.

---

## Load Forecast

### How It Works

Predicts electricity consumption using a statistical model based on historical patterns, adjusted for seasonality, temperature, and Czech holidays.

| Property | Value |
|----------|-------|
| Method | Weekly profile from last 28 days |
| Granularity | 15-minute intervals |
| Horizon | 7 days ahead |
| DB task | `25` (`pgep_tasks.task_definitions`) |
| Schedule | `every:3600seconds`, **currently disabled** (last run 2026-04-04) |
| Manual trigger | `POST /api/v2/tasks/run-load-forecast` |
| Storage | `pge_data.load_forecast` |
| Script | `pgepilot-service/scripts/load_forecast.php` |

### Seasonal Correction Factors

| Month | Factor | Reason |
|-------|--------|--------|
| January | 1.35 | Higher heating load |
| February | 1.25 | Heating |
| March | 1.10 | Transition |
| April | 1.00 | Baseline |
| May | 0.90 | Lower load |
| June | 0.80 | Summer |
| July | 0.70 | Lowest (vacation + no heating) |
| August | 0.75 | Summer |
| September | 0.90 | Transition |
| October | 1.05 | Heating starts |
| November | 1.20 | Heating |
| December | 1.30 | Heating + holidays |

### Temperature Correction

- **Heating threshold**: Below ~15°C, load increases (electric heating)
- **Cooling threshold**: Above ~28°C, load increases (air conditioning)
- Czech holiday detection adjusts profile (weekday → weekend pattern)

---

## Weather Forecast

### How It Works

Fetches weather data from [Open-Meteo](https://open-meteo.com) free API alongside PV forecast.

| Property | Value |
|----------|-------|
| Source | Open-Meteo API (free tier; DWD ICON minutely_15 for 0-48 h, Forecast API hourly for 3-16 d) |
| Variables | Temperature, cloudiness, GHI/DNI/DHI, precipitation, wind |
| Points | 15-min subset for 0-48 h + hourly to ~16 days |
| DB task | `26` (shared with PV OM -- a single run fetches weather + computes PV) |
| Schedule | `every:3600seconds` |
| Storage | `pge_data.weather_forecast` (keyed by GPS, not by CP) |

### API Call

```
GET https://api.open-meteo.com/v1/forecast
  ?latitude={LAT}&longitude={LON}
  &timezone=Europe/Prague
  &hourly=temperature_2m,cloud_cover,shortwave_radiation,precipitation,wind_speed_10m
```

---

## Adaptive Correction

### How It Works

Compares yesterday's forecast with actual production using Exponential Moving Average (EMA) to adjust future forecasts. Maintains separate EMA factors per source (`correction_factor` for forecast.solar branch, `correction_factor_om` for OpenMeteo branch).

| Property | Value |
|----------|-------|
| Method | EMA (Exponential Moving Average) |
| Alpha | 0.15 (smooth, slow adaptation) |
| Clamp range | **0.70 -- 1.50** (as implemented in `forecast_correction.php`) |
| Curtailment guard | Skip day if battery full AND export disabled (prevents negative bias) |
| DB task | `27` |
| Schedule | `daily:01:05` |
| Manual trigger | `POST /api/v2/tasks/run-forecast-correction` |
| Storage | `pge_data.forecast_correction_log` |
| Script | `pgepilot-service/scripts/forecast_correction.php` |

### Logic

```
For each enabled plant, for each source in {forecast_solar, openmeteo}:
  1. Get yesterday's source-specific PV forecast (predicted kWh)
  2. Get yesterday's actual production (from effective_history)
  3. Detect curtailment (battery full + export disabled) -> skip day
  4. ratio = actual / predicted
  5. new_correction = alpha * ratio + (1 - alpha) * old_correction
  6. clamp(new_correction, 0.70, 1.50)
  7. Store in forecast_correction_log
  8. Update pv_forecast_config.{correction_factor | correction_factor_om}
  9. Factor is applied to the next forecast run of the matching source
```

### Observed Behaviour (2026-04-18..20 production samples)

The OpenMeteo model systematically overshoots for the two real plants — both `va` and `kder_vel` have converged to `correction_factor_om = 0.70` (the lower clamp), implying the raw OM forecast is ~40 % high. See **Known Issues** for likely causes.

---

## Scheduling and Execution

Current production recurring execution is DB-backed, not crontab-based. Definitions live in the dedicated `pgep_tasks` database (not `pge_control`).

```
pgep_tasks.task_definitions (MariaDB)
  |
  | -> TaskManager (pgepilot-service, Specific/TaskManager/TaskManager.php)
  | -> http://pgepilot_jobmanager:5000/add_task
  | -> JobManager dispatches to worker by worker_type
  v
worker POST /execute-task { controllerFunction: "runPvForecastOm", ... }
  |
  v
TaskController::runPvForecastOm() etc. -> shells out to scripts/<name>.php -> pge_data tables
```

Note: the endpoint stored in each `task_definitions` row is `http://service.pgepilot.cz/execute-task`, and the controller to invoke is carried in `parameters.controllerFunction`. The `/api/v2/tasks/run-*` routes in `pgepilot-service/app/routes_pge_control.php` exist as **manual triggers** only — the scheduler does not call them.

### Manual Operations

```bash
# Trigger forecast.solar manually
curl -X POST http://pgepilot_service/api/v2/tasks/run-pv-forecast

# Trigger load forecast manually
curl -X POST http://pgepilot_service/api/v2/tasks/run-load-forecast

# Trigger Open-Meteo PV forecast manually
curl -X POST http://pgepilot_service/api/v2/tasks/run-pv-forecast-om

# One-off historical backfill of the OpenMeteo physical model
curl -X POST http://pgepilot_service/api/v2/tasks/run-pv-forecast-om-backfill

# Trigger correction manually
curl -X POST http://pgepilot_service/api/v2/tasks/run-forecast-correction
```

### Task Definitions in DB (`pgep_tasks.task_definitions`)

| Task ID | Name | Active | Schedule | Last Run (2026-04-21) | Notes |
|---------|------|--------|----------|----------------------|-------|
| 24 | PV Forecast (forecast.solar) | 0 | `every:3600seconds` | 2026-04-04 15:20 | Script retained; has not run for ~17 days |
| 25 | Load Forecast | 0 | `every:3600seconds` | 2026-04-04 15:20 | Script retained; has not run for ~17 days |
| 26 | PV Forecast OpenMeteo | 1 | `every:3600seconds` | same day | Active hourly production task |
| 27 | Forecast Correction | 1 | `daily:01:05` | same day | Active daily production task |
| 28 | Task Cleanup | 1 | `every:3600seconds` | same day | Maintenance task; often discussed together with forecast automation |

The old `pgepilot-js` scheduler is no longer canonical — the original `scheduledTasks` array and `/scheduled_tasks` / `/run_scheduled_task` endpoints are commented out in `pgepilot-js/jobmanager.js` (the migration comment reads: "migrated to task_definitions"). Current production recurring execution is represented entirely by these DB task definitions plus TaskManager + JobManager + worker execution.

---

## API Endpoints

### Get Forecast Data

```
GET /api/v2/collection-points/{code}/forecast
```

Returns combined response:
- PV forecast (6 days, 15min intervals)
- Load forecast (7 days, 15min intervals)
- Weather forecast (48 points)
- Comparison (forecast vs actual for recent days)
- History (correction log)

### Trigger Forecasts (manual triggers, not used by the scheduler)

```
POST /api/v2/tasks/run-pv-forecast             -- forecast.solar pipeline
POST /api/v2/tasks/run-load-forecast           -- load profile pipeline
POST /api/v2/tasks/run-pv-forecast-om          -- Open-Meteo physical PV model
POST /api/v2/tasks/run-pv-forecast-om-backfill -- Open-Meteo historical backfill
POST /api/v2/tasks/run-forecast-correction     -- adaptive EMA correction
```

These routes live in `pgepilot-service/app/routes_pge_control.php` (group `/api/v2`, around lines 4230-4270) and dispatch to `TaskController::runPvForecast` / `runLoadForecast` / `runPvForecastOm` / `runForecastCorrection`. The scheduler reaches the same controllers via the generic `/execute-task` entry point (see the **Scheduling and Execution** section).

---

## Frontend (PGE App)

Older UI notes referenced a dedicated `/predikce` page, but current `pge-app` `main` has no dedicated forecast route. The forecast view lives instead in the customer shell as [`src/views/customer/CustomerForecast.vue`](../pge-app/src/views/customer/CustomerForecast.vue), which fetches the combined `GET /api/v2/collection-points/{code}/forecast` response (PV + load + weather + correction history) and renders the 4 panels. Forecast data is also surfaced in dashboards and collection-point detail views. A forecast overlay in the main Charts page is still pending (Priority 2.6 in the roadmap).

---

## Database Tables

Forecast **data** lives in `pge_data`; forecast **configuration** lives in `pge_control.pv_forecast_config` (see schema above):

| DB | Table | Content | Key columns |
|----|-------|---------|-------------|
| `pge_data` | `pv_forecast` | PV production forecast (per plant, per timestamp) | `collection_point_id, ts_local, pv_w, pv_w_s1, pv_w_s2, cloud_cover_pct, clear_sky_index, confidence, source ∈ {forecast_solar, solcast, ml_model, manual, openmeteo}` |
| `pge_data` | `load_forecast` | Load consumption forecast (per plant, per timestamp) | `collection_point_id, ts_local, load_w, day_type, seasonal_factor, temp_correction, source` |
| `pge_data` | `weather_forecast` | Weather data (temperature, cloud, GHI/DNI/DHI, rain, wind) -- keyed by site GPS, not by CP | `latitude, longitude, ts_local, temperature_c, cloud_cover_pct, precipitation_mm, wind_speed_ms, ghi_wm2, dni_wm2, dhi_wm2, source` |
| `pge_data` | `forecast_correction_log` | EMA correction history (per plant, per source, per day) | `collection_point_id, source, date, predicted_kwh, actual_kwh, ratio, old_factor, new_factor` |
| `pge_control` | `pv_forecast_config` | Per-CP forecast configuration (see full schema in earlier section) | `id (=CP id), enabled, tier, forecast_source, strings_json, correction_factor, correction_factor_om, model_version, shade_enabled, snow_model` |

DDL for all four `pge_data.*` tables is **inline in the scripts** (`CREATE TABLE IF NOT EXISTS` at the top of each forecast script) rather than in a migrations directory.

---

## Open-Meteo Physical PV Model (Phase 1 in production)

> Full design specification: [`assets/open-meteo-fve-navrh.md`](assets/open-meteo-fve-navrh.md) (v1, 2026-03-30)

A self-hosted physical PV model using Open-Meteo as the weather data source, designed to complement or replace `forecast.solar`. **Phase 1 (MVP) is implemented and in production** as `pgepilot-service/scripts/pv_forecast_om.php` (~880 lines), driven hourly by task `26` (`runPvForecastOm`).

### Why

- forecast.solar is a paid API (Professional tier) -- cost per plant
- Own model allows customization (clipping, battery, multi-string)
- Open-Meteo provides raw irradiance data (GHI, DNI, DHI) for free
- Better control over forecast quality and calibration

### Implementation Phases

| Phase | What | Detail | Status |
|-------|------|--------|--------|
| **1 - MVP** | Physical model | GHI/DNI/DHI from Open-Meteo → POA transposition → T_cell → P_dc → P_ac | **Production** (pv_forecast_om.php) |
| **2 - Calibration** | Real vs predicted | Tune eta_balance, temperature model, clipping | Partial (`forecast_model_train.php` exists; systematic calibration not run) |
| **3 - Forecast quality** | Historical + Previous Runs API | Bias correction for D+1 and D+2 | Not yet |
| **4 - Satellite** | Satellite Radiation API | Near-real irradiance reference for nowcasting | Not yet |

### Physical Model Algorithm (Phase 1 -- as implemented)

The live implementation uses the Hay-Davies anisotropic diffuse model (the simple isotropic formula below is kept for illustration only).

```
Inputs:
  - GPS coordinates (latitude, longitude)
  - Panel config: tilt (β), azimuth (γ_p), count, Wp, γ_pmp (temp coeff), NOCT
  - System: eta_balance (DC→AC efficiency), P_ac_limit (inverter limit)
  - Multi-string config from pv_forecast_config.strings_json

Weather from Open-Meteo (per fetch strategy below):
  - GHI = shortwave_radiation
  - DNI = direct_normal_irradiance
  - DHI = diffuse_radiation
  - T_air = temperature_2m
  - V_wind = wind_speed_10m

Step 1: Solar position (zenith θ_z, azimuth γ_s) via Spencer formulas from GPS + timestamp
Step 2: Angle of incidence on panel
         cos(θ_i) = cos(θ_z)·cos(β) + sin(θ_z)·sin(β)·cos(γ_s - γ_p)
Step 3: POA irradiance (Plane of Array) -- Hay-Davies anisotropic model
         POA_beam    = DNI × max(0, cos(θ_i))
         A_i         = DNI / I_sc                                -- anisotropy index
         POA_diffuse = DHI × [A_i · cos(θ_i)/cos(θ_z) + (1-A_i)·(1+cos(β))/2]
         POA_ground  = GHI × 0.2 × (1 - cos(β)) / 2
         POA_total   = POA_beam + POA_diffuse + POA_ground
Step 4: Cell temperature (NOCT model)
         T_cell = T_air + (NOCT - 20) / 800 × POA_total
Step 5: DC power
         P_dc = P_stc_total × (POA_total / 1000) × (1 + γ_pmp × (T_cell - 25))
Step 6: AC power
         P_ac = min(P_dc × eta_balance, P_ac_limit)
Step 7: v2_factored extras (if pv_forecast_config.model_version='v2_factored'):
         f_shade  -- horizon shadow attenuation per hour (if shade_enabled=1)
         f_snow   -- snow-cover attenuation when applicable (if snow_model=1)
         f_cloud  -- non-linear cloud opacity correction
         f_drift  -- azimuthal/seasonal drift
         P_ac *= f_shade · f_snow · f_cloud · f_drift
Step 8: Apply correction_factor_om from pv_forecast_config (EMA from task 27)
Step 9: Daily energy = ∫ P_ac(t) dt
```

### Weather Fetch Strategy (as implemented)

- **0-48 hours horizon**: DWD ICON `minutely_15` (native 15-minute resolution for Central Europe)
- **3-16 days horizon**: generic Forecast API at hourly resolution
- The two are stitched together in `pv_forecast_om.php`; 15-minute rows populate the near horizon in `pv_forecast` while hourly rows cover the extended horizon.

### Open-Meteo Endpoints

| Endpoint | Use | Granularity |
|----------|-----|-------------|
| Forecast API hourly | Daily production forecast, 1-16 days | 1 hour |
| DWD ICON minutely_15 | Intraday forecast for Czech Republic, 0-48h | **15 min** (native for Central Europe) |
| Historical Weather API (archive) | Long backfill (2+ years), reanalysis | 1 hour |
| Historical Forecast API | Training ML models, forecast quality analysis | 1 hour |
| Previous Runs API | Bias correction (what yesterday's forecast said about today) | 1 hour |
| Satellite Radiation API | Near-real irradiance reference (Phase 4) | variable |

### Planned Database Tables

```sql
-- Weather forecast snapshots
meteo_forecast_hourly (
  site_id, fetched_at_utc, target_time_local, source_endpoint,
  ghi_wm2, dni_wm2, dhi_wm2, gti_ref_wm2,
  temperature_2m_c, wind_speed_10m_ms, cloud_cover_pct
)

-- Historical actual weather (reanalysis)
meteo_actual_hourly (
  site_id, valid_time_local, dataset,
  ghi_wm2, dni_wm2, dhi_wm2,
  temperature_2m_c, wind_speed_10m_ms, cloud_cover_pct
)

-- Actual PV production (for calibration)
pv_actual (
  site_id, valid_time_local, power_w, energy_wh,
  inverter_limit_w, curtailment_flag
)
```

### Open-Meteo Limits

| Tier | Calls/day | Calls/hour | Commercial | Historical |
|------|-----------|------------|------------|------------|
| Free | 10,000 | 5,000 | No | Limited |
| Standard | ~1M/month | -- | Yes | No |
| Professional | ~5M/month | -- | Yes | Yes (incl. Previous Runs) |

For production with multiple plants, Professional tier is recommended.

### Important Design Decisions

1. **Use GHI + DNI + DHI, not just GTI** -- Open-Meteo GTI uses fixed 20% albedo and isotropic sky. Own transposition is more accurate.
2. **Separate forecast from actual** -- never mix forecast data with historical actual in one dataset without labeling the source.
3. **Backfill in monthly batches** -- Open-Meteo charges requests with 10+ variables or 2+ weeks as multiple API calls.
4. **No ML in Phase 1** -- start with physical model, add bias correction in Phase 2-3, ML in Phase 4+.
5. **DWD ICON for Czech Republic** -- native 15min solar data for Central Europe, no interpolation needed.

---

## Known Issues (as of 2026-04-21)

- **Coverage**: PV forecast is enabled for only 2 real plants (`va`, `kder_vel`) out of 47 active CPs, plus 3 synthetic demo rows (`demo_kd`, `demo_rad`, `demo_zs`). Expanding coverage is Priority 1.5 in the roadmap.
- **OpenMeteo model systematically overshoots**: both real plants have converged to `correction_factor_om = 0.70` (the lower EMA clamp), implying the raw OM prediction is ~40 % high. Likely root causes: `eta_balance` too generous, shade horizon not modelled (`kder_vel` has `shade_enabled=0`), or inverter clipping not captured. Candidates for Phase 2 calibration.
- **forecast.solar API key is hardcoded** in `pv_forecast.php` (should be in env var or config table).
- **Tasks `24` (forecast.solar) and `25` (load forecast) have been disabled since 2026-04-04** -- their scripts still exist but no fresh data is being produced. `va` and `kder_vel` rows still have `forecast_source='forecast_solar'` in config, but in practice the OpenMeteo pipeline (task `26`) is what serves them.
- **Load forecast for real plants has never produced continuous data**: `load_forecast` contains only `source='demo_profile'` (seed rows) and `source='profile'` rows written by task 25, all of which stopped on 2026-04-05. Task 25 re-enabling is required for the internal profiling pipeline to resume.
- **No forecast overlay in main Charts page** (dedicated `/predikce` route removed from pge-app `main`; forecast now consumed via `CustomerForecast.vue` and backend APIs). Priority 2.6 in roadmap.
- **Dead reference**: the previous revision of this document pointed to `zadani/sbc pgepilot/Predikce/open-meteo-fve-navrh.md`, which lived outside the git repo (OneDrive). The design doc has been copied into this repo at [`assets/open-meteo-fve-navrh.md`](assets/open-meteo-fve-navrh.md).
- **No migrations for forecast tables**: DDL is inline `CREATE TABLE IF NOT EXISTS` inside the forecast scripts; there is no versioned migration history for `pv_forecast`, `load_forecast`, `weather_forecast`, `forecast_correction_log`.
- **Only task 24 endpoint is referenced as "trigger" in the overview** -- `TaskController` actually exposes a fifth endpoint `runPvForecastOmBackfill` (route `/api/v2/tasks/run-pv-forecast-om-backfill`) for ad-hoc historical backfill of the OpenMeteo pipeline, and a separate `forecast_model_train.php` script (~27 KB) exists for offline model training; neither is surfaced in the task schedule.
