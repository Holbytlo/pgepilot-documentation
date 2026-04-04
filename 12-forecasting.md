# 12 -- Forecasting System

> PV production forecast, load forecast, weather data, adaptive correction, and planned Open-Meteo physical model.
> Last updated: 2026-04-04

---

## Overview

PgePilot runs three independent forecast pipelines plus an adaptive correction layer. All are managed by the JobManager scheduler (not crontab). Currently only 2 plants have PV forecast enabled.

| Pipeline | Source | Granularity | Horizon | Schedule | Status |
|----------|--------|-------------|---------|----------|--------|
| PV forecast | forecast.solar Professional | 15 min | 6 days | Hourly at :17 | Production (2 plants) |
| Load forecast | Internal profiling | 15 min | 7 days | Hourly at :23 | Production (all plants) |
| Weather forecast | Open-Meteo API | 48 points | ~2 days | With PV forecast | Production |
| Adaptive correction | EMA (alpha=0.15) | Daily | -- | Daily at 1:05 | Production |

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
| Schedule | Every hour at minute :17 |
| Trigger | `POST /api/v2/tasks/run-pv-forecast` |
| Storage | `pge_data.pv_forecast` |
| Config table | `pge_control.pv_forecast_config` |

### Currently Enabled Plants

Only **2 out of 35** collection points have PV forecast configured:

| Plant | Status |
|-------|--------|
| VA | Enabled |
| Kder Veltrusy | Enabled |

**Expanding to more plants** is Priority 1.5 in the roadmap -- requires adding rows to `pv_forecast_config` with panel parameters (azimuth, tilt, capacity per string).

### pv_forecast_config Table

```sql
-- Key columns:
collection_point_id  CHAR(36)
latitude             DECIMAL(10,7)
longitude            DECIMAL(10,7)
declination          INT           -- panel tilt (degrees)
azimuth              INT           -- panel azimuth (degrees, 0=north, 180=south)
kwp                  DECIMAL(6,2)  -- peak power per string (kWp)
enabled              TINYINT(1)
```

---

## Load Forecast

### How It Works

Predicts electricity consumption using a statistical model based on historical patterns, adjusted for seasonality, temperature, and Czech holidays.

| Property | Value |
|----------|-------|
| Method | Weekly profile from last 28 days |
| Granularity | 15-minute intervals |
| Horizon | 7 days ahead |
| Schedule | Every hour at minute :23 |
| Trigger | `POST /api/v2/tasks/run-load-forecast` |
| Storage | `pge_data.load_forecast` |

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
| Source | Open-Meteo API (free tier) |
| Variables | Temperature, cloudiness, GHI, precipitation, wind |
| Points | 48 forecast points |
| Storage | `pge_data.weather_forecast` |

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

Compares yesterday's forecast with actual production using Exponential Moving Average (EMA) to adjust future forecasts.

| Property | Value |
|----------|-------|
| Method | EMA (Exponential Moving Average) |
| Alpha | 0.15 (smooth, slow adaptation) |
| Clamp range | 0.50 -- 1.80 (correction never exceeds these bounds) |
| Schedule | Daily at 1:05 |
| Trigger | `POST /api/v2/tasks/run-forecast-correction` |
| Storage | `pge_data.forecast_correction_log` |
| Script | `forecast_correction.php` |

### Logic

```
For each enabled plant:
  1. Get yesterday's PV forecast (predicted kWh)
  2. Get yesterday's actual production (actual kWh)
  3. ratio = actual / predicted
  4. new_correction = alpha * ratio + (1 - alpha) * old_correction
  5. clamp(new_correction, 0.50, 1.80)
  6. Store in forecast_correction_log
  7. Apply correction to next forecast
```

---

## Scheduling and Execution

All forecast tasks are managed by the **JobManager scheduler** (Node.js, setInterval 60s), NOT by crontab.

```
JobManager scheduler (checks time every 60s)
  |
  | :17 -> POST http://pgepilot_service/api/v2/tasks/run-pv-forecast
  | :23 -> POST http://pgepilot_service/api/v2/tasks/run-load-forecast
  | 1:05 -> POST http://pgepilot_service/api/v2/tasks/run-forecast-correction
  v
Service (PHP) -> runs forecast PHP script -> stores result in pge_data tables
```

### Manual Operations

```bash
# Check scheduled tasks status
curl http://pgepilot_jobmanager:5000/scheduled_tasks

# Manually trigger PV forecast
curl -X POST http://pgepilot_jobmanager:5000/run_scheduled_task -d '{"name":"pv_forecast"}'

# Manually trigger load forecast
curl -X POST http://pgepilot_jobmanager:5000/run_scheduled_task -d '{"name":"load_forecast"}'

# Manually trigger correction
curl -X POST http://pgepilot_jobmanager:5000/run_scheduled_task -d '{"name":"forecast_correction"}'
```

### Task Definitions in DB

| Task ID | Name | Active | Notes |
|---------|------|--------|-------|
| 24 | PV Forecast (forecast.solar) | 0 | Runs via JobManager scheduler, not task system |
| 25 | Load Forecast | 0 | Runs via JobManager scheduler |
| 26 | PV Forecast OpenMeteo | 0 | Alternative source, not yet implemented |
| 27 | Forecast Correction | 0 | Runs via JobManager scheduler |

These task definitions exist but are disabled (active=0) because the forecasts run through the JobManager scheduler mechanism, not the standard task execution pipeline.

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

### Trigger Forecasts (internal, called by JobManager)

```
POST /api/v2/tasks/run-pv-forecast
POST /api/v2/tasks/run-load-forecast
POST /api/v2/tasks/run-forecast-correction
```

---

## Frontend (PGE App)

The `/predikce` page (admin only) shows forecast data in 3 tabs:

| Tab | Content |
|-----|---------|
| PV Production | forecast.solar prediction chart, actual vs predicted |
| Consumption | Load forecast chart, historical profile |
| Balance + Weather | Net balance (PV - load), weather overlay (temperature, cloud, GHI) |

Component: `src/views/Forecast.vue`

---

## Database Tables

All forecast data is stored in `pge_data` database:

| Table | Content |
|-------|---------|
| `pv_forecast` | PV production forecast (per plant, per timestamp) |
| `load_forecast` | Load consumption forecast (per plant, per timestamp) |
| `weather_forecast` | Weather data (temperature, cloud, GHI, rain, wind) |
| `forecast_correction_log` | EMA correction history (per plant, per day) |

---

## Planned: Open-Meteo Physical PV Model

> Full specification: `zadani/sbc pgepilot/Predikce/open-meteo-fve-navrh.md`

A planned upgrade to replace or complement forecast.solar with a self-hosted physical PV model using Open-Meteo as the weather data source.

### Why

- forecast.solar is a paid API (Professional tier) -- cost per plant
- Own model allows customization (clipping, battery, multi-string)
- Open-Meteo provides raw irradiance data (GHI, DNI, DHI) for free
- Better control over forecast quality and calibration

### Implementation Phases

| Phase | What | Detail |
|-------|------|--------|
| **1 - MVP** | Physical model | GHI/DNI/DHI from Open-Meteo → POA transposition → T_cell → P_dc → P_ac |
| **2 - Calibration** | Real vs predicted | Tune eta_balance, temperature model, clipping |
| **3 - Forecast quality** | Historical + Previous Runs API | Bias correction for D+1 and D+2 |
| **4 - Satellite** | Satellite Radiation API | Near-real irradiance reference for nowcasting |

### Physical Model Algorithm (Phase 1)

```
Inputs:
  - GPS coordinates (latitude, longitude)
  - Panel config: tilt (β), azimuth (γ_p), count, Wp, γ_pmp (temp coeff), NOCT
  - System: eta_balance (DC→AC efficiency), P_ac_limit (inverter limit)

Weather from Open-Meteo:
  - GHI = shortwave_radiation
  - DNI = direct_normal_irradiance
  - DHI = diffuse_radiation
  - T_air = temperature_2m
  - V_wind = wind_speed_10m

Step 1: Solar position (zenith θ_z, azimuth γ_s) from GPS + timestamp
Step 2: Angle of incidence on panel
         cos(θ_i) = cos(θ_z)·cos(β) + sin(θ_z)·sin(β)·cos(γ_s - γ_p)
Step 3: POA irradiance (Plane of Array)
         POA_beam    = DNI × cos(θ_i)
         POA_diffuse = DHI × (1 + cos(β)) / 2
         POA_ground  = GHI × 0.2 × (1 - cos(β)) / 2
         POA_total   = POA_beam + POA_diffuse + POA_ground
Step 4: Cell temperature
         T_cell = T_air + (NOCT - 20) / 800 × POA_total
Step 5: DC power
         P_dc = P_stc_total × (POA_total / 1000) × (1 + γ_pmp × (T_cell - 25))
Step 6: AC power
         P_ac = min(P_dc × eta_balance, P_ac_limit)
Step 7: Daily energy = ∫ P_ac(t) dt
```

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

## Known Issues

- PV forecast only for 2/35 plants (VA, Kder Veltrusy) -- need to expand pv_forecast_config
- forecast.solar API key is hardcoded (should be in env var or config table)
- Task definitions 24-27 exist but are disabled (forecasts run via JobManager scheduler mechanism)
- No forecast overlay in main Charts page (only separate /predikce page) -- Priority 2.6 in roadmap
- Open-Meteo physical model is designed but NOT implemented yet
