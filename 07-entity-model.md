# 07 -- Entity Model

> Domain model for the Energity/PgePilot system: new (pge_control) and legacy (pgepilot) schemas.
> Last updated: 2026-04-04

---

## Critical Rule: Database Separation

```
Legacy databases (pgepilot, pgepilot_data) are NEVER modified.
New entity model goes into SEPARATE databases:

  pge_control  -- cp_* entities, commands, events, configs, realtime
  pge_data     -- new time series (power_1m, energy_15m, forecast)
  pgep_tasks   -- task management (shared, no changes)

Legacy and new systems coexist. Legacy is NOT disabled until new system is fully operational.
NO ALTER TABLE on legacy tables!
```

---

## New Entity Model (pge_control) -- CANONICAL

```
Control Domain (portfolio/tenant, e.g. "Energity demo", "Obec Citoliby")
  +-- Collection Point (site/location = today's Plant, e.g. "Citoliby hasicarna")
        +-- Device (logical unit: PLANT, BOILER, EVSE, POOL, HEATING)
        |     +-- Machine (physical HW: INVERTER, BATTERY, METER, BOILER_CTRL)
        |           +-- Control Point (signal: grid.pW, battery.socPct, exportLimitW)
        +-- Connector (integration: GOODWE_SEMS, SOLAX_CLOUD, SUPLA_RELAY, SMARTBOX_RPC)
        +-- Binding (routing: Control Point <-> Connector, with transform and priority)
```

### Production State (verified from live DB, 2026-04-04)

| Table | Count |
|-------|-------|
| cp_control_domains | **32** |
| cp_collection_points | **35** |
| cp_devices | 26 |
| cp_machines | 40 |
| cp_connectors | **39** |
| cp_users | **32** |
| cp_user_grants | 31 |
| realtime_state | 23 |
| pv_forecast_config | 2 enabled |
| alert_log | 116 |
| api_response_cache | 2166 |
| connector_cache_config | 18 |
| connector_config | 7 |
| cp_automation_rules | 31 |
| cp_state_history | 1342 |
| data_column_meta | 20 |
| data_source_registry | 39 |
| sb_schema_meta | 2 |

Total: **27 tables** in pge_control.

### Tree Rules

- Tree is two-level: Device -> Machine (no further recursion)
- Control Point is a named signal with kind: telemetry / state / setpoint / command
- Connector has aggregationLevel: DEVICE (vendor API) or MACHINE (SmartBox/relay)
- Binding has priority, transform_json (scale/invert/clamp), enabled flag

### Naming Convention

- `cp_` prefix on tables = "control-plane" namespace (NOT abbreviation for Collection Point)
- `cp_collection_points` = Collection Point (site)
- `cp_control_points` = Control Point (signal on a Machine)
- `{collection_point_code}_power_1m` = time series table per Collection Point

---

## Key Tables (DDL Summary)

### cp_control_domains
```sql
CREATE TABLE cp_control_domains (
  id         CHAR(36) PRIMARY KEY,    -- UUID
  name       VARCHAR(255) NOT NULL,
  slug       VARCHAR(100) NOT NULL UNIQUE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### cp_collection_points
```sql
CREATE TABLE cp_collection_points (
  id                  CHAR(36) PRIMARY KEY,
  domain_id           CHAR(36) NOT NULL,        -- FK -> cp_control_domains
  code                VARCHAR(50) NOT NULL UNIQUE,  -- stable code (va, bd41, sicak)
  label               VARCHAR(255) NOT NULL,
  timezone            VARCHAR(50) DEFAULT 'Europe/Prague',
  address             VARCHAR(500),
  latitude            DECIMAL(10,7),
  longitude           DECIMAL(10,7),
  pv_capacity_w       INT UNSIGNED,
  battery_capacity_wh INT UNSIGNED,
  status              ENUM('active','inactive','provisioning') DEFAULT 'active'
);
```

### cp_devices
```sql
CREATE TABLE cp_devices (
  id                  CHAR(36) PRIMARY KEY,
  collection_point_id CHAR(36) NOT NULL,        -- FK -> cp_collection_points
  type                ENUM('PLANT','BOILER','EVSE','POOL','HEATING','LIGHTING','GENERIC'),
  label               VARCHAR(255) NOT NULL,
  priority            SMALLINT UNSIGNED DEFAULT 0,
  max_power_w         INT UNSIGNED,
  enabled             TINYINT(1) DEFAULT 1
);
```

### cp_machines
```sql
CREATE TABLE cp_machines (
  id        CHAR(36) PRIMARY KEY,
  device_id CHAR(36) NOT NULL,                -- FK -> cp_devices
  type      ENUM('INVERTER','BATTERY','METER','BOILER_CTRL','RELAY','EVSE_CTRL'),
  sn        VARCHAR(100),                     -- serial number
  model     VARCHAR(255),
  rated_ac_w            INT UNSIGNED,
  max_export_w          INT UNSIGNED,
  max_import_w          INT UNSIGNED,
  rated_batt_charge_w   INT UNSIGNED,
  rated_batt_discharge_w INT UNSIGNED,
  battery_capacity_wh   INT UNSIGNED
);
```

### realtime_state
```sql
CREATE TABLE realtime_state (
  collection_point_id CHAR(36) PRIMARY KEY,
  pv_w           INT,            -- PV production (W)
  load_w         INT,            -- Load consumption (W)
  grid_w         INT,            -- Grid power (+ export, - import)
  battery_w      INT,            -- Battery (+ charge, - discharge)
  soc_pct        DECIMAL(5,2),   -- State of Charge (%)
  severity       TINYINT DEFAULT 0,  -- 0=OK, 1-4 escalating
  updated_at     DATETIME
);
```

### connector_config (added 2026-04-04)
```sql
-- columns added to existing table:
realtime_enabled  TINYINT(1) DEFAULT 1    -- enable/disable realtime collection
backfill_enabled  TINYINT(1) DEFAULT 1    -- enable/disable backfill
```

---

## Legacy Entity Model (pgepilot DB)

```
User -> Customer -> Plant (code, EANd, pvCapacity, batteryCapacity)
  Plant -> Machine[] (sn, type_id -> machine_types)
  Plant -> relay_groups[] (= Control Points: BOILER/WALLBOX/POOL/HEATING)
  |          +-- relays[] (Supla cloud, server_code, channel_id)
  |          +-- relay_rule_template_id (control strategy)
  Plant -> plants_action (active strategy, health check config)
  Plant -> plant_realtime_status (live: acpower, feedin, dc, battery, soc, yield)
  Plant -> v_plant_latest_status (VIEW: severity 0-4, state, reason)
  Plant -> event_log[] (alerts, history)
  Plant -> plant_state_history[] (state timeline)
  Plant -> plant_notifications[] (email notification config)
```

---

## Legacy <-> New Mapping

| Legacy (pgepilot) | New (pge_control) | Notes |
|-------------------|-------------------|-------|
| plants | cp_collection_points | 1:1, backfill done |
| machines | cp_machines | 1:1, via cp_devices |
| relay_groups | cp_devices (BOILER/EVSE...) | Extended with device types |
| relay_rule_templates | TOU engine (on SmartBox) | Legacy strategies vs TOU |
| plant_realtime_status | realtime_state | New format (pv_w, load_w, grid_w) |
| event_log | cp_events | Currently empty in new system |
| rpc_commands | cp_commands | 2 records in new system |

---

## Severity Levels (Monitoring)

| Level | Description | Example |
|-------|-------------|---------|
| 0 | Everything OK | Plant operating normally |
| 1 | Suspicious -- needs verification | Unusually low production |
| 2 | Error -- non-critical | API timeout, temporary data gap |
| 3 | Partial operation / limited data | Some inverter offline |
| 4 | Outage / critical error | Plant not responding |

---

## Control Strategies (Legacy relay_rule_templates)

| ID | Name | Strategy | SoC Up | SoC Down | Feed Up W | Feed Down W |
|----|------|----------|--------|----------|-----------|-------------|
| 1 | Standard operation | DEFAULT | 80% | 40% | 2000 | 100 |
| 2 | Summer - full battery | FORCE_ON_IF_SOC_FULL | 80% | 70% | -- | -- |
| 99 | Disabled | -- | -- | -- | -- | -- |

**Logic**:
- SoC > up AND feed > feed_up -> TURN ON appliance
- SoC < down OR feed < feed_down -> TURN OFF appliance
- Strategy is per-plant (plants_action), can be overridden per-group (relay_groups.relay_rule_template_id)

---

## Time Series Databases

### pge_data (new)
Per Collection Point (using `code` as prefix):
- `{code}_power_1m` -- 1-minute power readings (W)
- `{code}_energy_15m` -- 15-minute energy aggregation (Wh)
- `{code}_export_alloc_15m` -- 15-minute export allocation

Global:
- `pv_forecast` -- PV forecast data
- `load_forecast` -- Load forecast data
- `weather_forecast` -- Weather forecast data
- `forecast_correction_log` -- Adaptive correction history

Total: 81 tables.

### pgepilot_data (legacy, migrated from pgedata.cz)
- `{code}_power_5m` -- 5-minute power readings
- `{code}_energy_5m` -- 5-minute energy readings

Total: 55 tables.
