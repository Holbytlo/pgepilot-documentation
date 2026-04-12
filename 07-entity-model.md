# 07 -- Entity Model

> Domain model for the Energity/PgePilot system: new (pge_control) and legacy (pgepilot) schemas.
> Last updated: 2026-04-12

---

## Critical Rule: Database Separation

```
Legacy databases (pgepilot, pgepilot_data) are NEVER modified.
New entity model goes into SEPARATE databases:

  pge_control  -- cp_* entities, commands, events, configs, realtime
  pge_data     -- time-series datasets (raw/source, derived/reporting, forecast)
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
| sems_plant_discovery | **228** (full SEMS plant catalog, in_pgepilot flag) |

Total: **28 tables** in pge_control.

### Tree Rules

- Tree is two-level: Device -> Machine (no further recursion)
- Control Point is a named signal with kind: telemetry / state / setpoint / command
- Connector has aggregationLevel: DEVICE (vendor API) or MACHINE (SmartBox/relay)
- Binding has priority, transform_json (scale/invert/clamp), enabled flag

### SmartBox Identity Bundle

For SmartBox provisioning the canonical identity bundle is:

- `sb_id` = physical box in `sb-manager`
- `collection_point_id` = site in `cp_collection_points`
- `device_id` = logical SmartBox device in `cp_devices`
- `machine_ids[]` = physical machines in `cp_machines`
- `smartbox_id` = SmartBox auth identity in `cp_connector_auth`
- `connector_id` = SmartBox RPC connector in `cp_connectors`

Rules:

- `smartbox_id` is the SmartBox identity on the wire
- `collection_point_id` and `device_id` are authoritative server-side routing keys
- `plantId` is not a SmartBox identity field anymore
- no new provisioning writes go to legacy `pgepilot.plants` or `pgepilot.machines`
- `login` is credential only; it must not be treated as the box identity

### Naming Convention

- `cp_` prefix on tables = "control-plane" namespace (NOT abbreviation for Collection Point)
- `cp_collection_points` = Collection Point (site)
- `cp_control_points` = Control Point (signal on a Machine)
- `{collection_point_code}_power_1m` = common physical table name for 1-minute raw/source history where available
- `tpl_power_1m` = template table used by CREATE TABLE LIKE for new plants
- `{collection_point_code}_energy_15m` = canonical 15-minute energy table with lineage columns
- physical table names still follow per-CP convention, but API/runtime resolution is driven by `dataset_registry` + `data_source_registry`, not by name guessing alone

### Dataset Ownership and Lineage

- raw/source datasets belong to an owner entity (`collection_point`, `device`, or `machine`) and a concrete connector instance
- derived/reporting datasets belong to the owner entity and use-case, but each row should still preserve connector lineage
- `dataset_registry` describes what a dataset is, who owns it, its granularity, and physical storage
- `data_source_registry` describes which dataset should be used for a given runtime scope (`effective_history`, `raw_history`, `forecast`, `effective_realtime`, ...)

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
  battery_w      INT,            -- Battery (+ discharge, - charge)
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
Per collection point (using `code` as prefix) the database currently contains a mix of canonical/source storage and reporting views or derived datasets:

- raw/source history:
  - `tpl_power_1m` -- template table for high-resolution source history
  - `{code}_power_1m` -- canonical minute history fed by GoodWe backfill, SmartBox push/pull, or realtime recording
  - `{code}_power_rt` -- realtime rolling power series
- derived/reporting history:
  - `{code}_power_bf` -- optional physical 5-minute-style reporting table; when absent, API resolves the same profile from `{code}_power_1m`
  - `{code}_power_1h` / `{code}_power_1d` -- coarser reporting aggregations or virtual reporting profiles
  - `{code}_energy_15m` -- canonical 15-minute energy aggregation with source lineage columns
  - `{code}_export_alloc_15m` -- 15-minute export allocation
- forecast:
  - `pv_forecast`
  - `load_forecast`
  - `weather_forecast`
  - `forecast_correction_log`

The important rule is not the suffix alone, but the dataset role:

- raw/source datasets are the source of truth from connectors
- derived/reporting datasets are serving datasets for UI/reporting/analytics
- API resolution for history/reporting goes through `dataset_registry` and `data_source_registry`

Total: 81+ tables (growing as new plants are added).

### pge_control -- Discovery Tables (added 2026-04-04)

- `sems_plant_discovery` -- catalog of all 228 SEMS plants discovered via getPowerStationList(). Contains `in_pgepilot` flag indicating which plants are already registered in PgePilot.

### pgepilot_data (legacy, migrated from pgedata.cz)
- `{code}_power_5m` -- 5-minute power readings
- `{code}_energy_5m` -- 5-minute energy readings

Total: 55 tables.
