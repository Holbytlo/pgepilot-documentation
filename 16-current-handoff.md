# 16 -- Current Handoff

> Short operational handoff for the next chat/window.
> Last updated: 2026-04-12

---

## Live Snapshot

This file exists because the production state changed across multiple chats on the same day and follow-up work must continue from one consistent handoff.

Current verified state:

- `pgepilot_service` runtime: `main@1727f60`
- `pgepilot_worker1/2/3` runtime: `main@1727f60`
- `pgepilot_servicedesk:/home/app2/pge-app`: `main@3d7e6bb`
- `sb-manager` runtime on VPS: `main@21df16b`
- local SmartBox source repo reference: `sb/devva@45a2fc0`
- public `app.pgepilot.cz` bundle: `index-lGjKNcFm.js` + `index-DfwOyOjc.css`
- `task_definitions`: `18=active`, `19=active`, `20=disabled`, `21=disabled`, `22=active`, `23=active`

Important:

- the history cutover and the SmartBox/auth follow-up were not the same deploy
- service + workers are uniform again on `1727f60`
- SB1, SB4, and SB7 now use separate SmartBox identity bundles; the old shared auth row is legacy residue only
- audited key runtime file hashes on SB1, SB4, and SB7 currently match the local `sb/devva@45a2fc0` content
- auth remains a separate topic; do not mix SmartBox auth changes into history debugging unless auth is the direct root cause

---

## What Is Already Done

- canonical history now targets `pge_data`
- GoodWe backfill writes `pge_data.{code}_power_1m`
- SmartBox `smartboxSendData` is mapped into canonical `power_1m` / reported `energy_15m`
- `power_bf` is treated as a reporting profile over canonical history
- `task 22` (`recordRealtimeToHistory`) was reactivated and writes current `realtime_state` into `power_1m`
- `task 23` computes `energy_15m` from `power_1m`
- web app already uses the new history `usage` contract
- GoodWe historical backfill now ingests one day at a time instead of one 7-day API payload per plant
- reverse backfill now stores a `plant_config.goodwe_reverse_empty_before_date` marker when the chunk immediately before current `min_date` is empty, so plants do not retry the same empty reverse window forever
- `task 18` has stayed clean in the later verification window: last 6h snapshot on `2026-04-12 22:20 CEST` showed `1205 completed`, `0 failed`
- `task 19` is also healthy in the same window: `1215 completed`, `9 sent`, `0 failed`
- SB1, SB4, and SB7 now have distinct `login + smartbox_id + collection_point_id + device_id` mappings generated through `sb-manager -> pgepilot-service`
- current SmartBox runtime expects canonical identity fields; legacy `plantId` is no longer the source of truth for provisioning
- SmartBox provisioning now enforces UUID validation in both `sb-manager` and `pgepilot-service`
- bad live IDs on `sb1 relay` and `sb7 inverter` were repaired end-to-end on `2026-04-12 23:16 CEST`
- `sb-manager -> pgepilot-service` cloud sync was re-run successfully for SB1 and SB7 after the repair

---

## What Is Still Open

### 1. Continue Monitoring Task 18 / Task 19

The previous hard failure mode was confirmed and fixed:

- root cause was a general GoodWe backfill issue, not one broken plant
- worker PHP hit `Allowed memory size of 536870912 bytes exhausted` while decoding a large `GetStationHistoryDataChart` response
- some plants also retried the same empty reverse chunk forever because the system did not remember that history already ended there

Deployed fix on `2026-04-12`:

- commit `131080f`: request only canonical history targets needed for `power_1m`
- commit `a30c78c`: ingest GoodWe history day-by-day and store a reverse-empty stop marker in `plant_config`

Post-deploy verification:

- manual canary for `BD41` (`e7f56834-1e0d-4a1a-b5dd-1438ff332e7d`) completed cleanly with `status=OK`, `processed_days=7`, `days_with_data=7`, `total_datapoints_saved=10045`
- manual canary for `FVE_DF55` (`df559764-d5f3-4101-a3cb-cc1b7d1831f3`) first returned clean empty reverse window, then second run returned `reverse_exhausted`
- `task_definition_id = 18` had `0` failed rows after `2026-04-12 17:40:00`
- later live snapshot at `2026-04-12 22:20 CEST`: `task 18 = 1205 completed / 0 failed` in the last 6h
- later live snapshot at `2026-04-12 22:20 CEST`: `task 19 = 1215 completed / 9 sent / 0 failed` in the last 6h
- no new Apache fatal/OOM entries were observed after deploy

This should now be treated as stabilized but still worth watching for a few scheduler cycles.

### 2. SmartBox Identity Validation Follow-up

The main provisioning gap is now fixed:

- `sb-manager/src/routes/admin.js` validates canonical UUID fields (`collectionPointId`, `smartboxId`, `deviceId`, `masterMachineId`, `machineIds`, nested `machine_id`)
- `pgepilot-service/app/smartbox_auth.php` enforces the same UUID rules on the provisioning endpoint, so direct API calls cannot bypass the guard
- direct live probe to `POST /api/v2/smartbox-auth/provision` now confirms invalid `machine_id` is rejected with `HTTP 400`
- repaired live machine IDs:
  - `sb1 relay`: `8528b7a4-b3e9-4518-9d02-5faf395927c7`
  - `sb7 inverter`: `ceefa067-dc8a-463c-b3ca-99b0b17dc510`
- repaired locations:
  - `pge_control.cp_machines`
  - `pge_control.cp_connector_auth.machine_ids_json`
  - `pge_control.cp_connectors.config_json`
  - `pgepilot.rpc_kv_chlum_zs`
  - `sb-manager` SQLite (`sb_devices.provisioning_machines_json`, `sb_cloud_identities.*machine*`)
  - live box configs on SB1 and SB7 (`rpc_client_config.yaml`, `smartbox_config.yaml`, `devices_config.yaml`)

Remaining follow-up:

- `sb13` still has its own invalid historical `machine_id` in live state and should be repaired in the separate sb13 thread before reprovisioning that device again

### 3. Auth context

Auth was handled in a different chat.

For this handoff:

- do not treat auth as part of the history-cutover task
- do not silently change auth contracts while debugging history
- if a task is about SB payload consistency, stay in SB/service payload scope unless auth is explicitly the bug
- if a task is about new SmartBox provisioning, use the `cp_*` identity model (`collection_point_id`, `device_id`, `smartbox_id`, `machine_id`) and do not write new legacy `plants/machines` assumptions back into docs or code

---

## Read This First

For cloud/history work:

1. [05-infrastructure.md](05-infrastructure.md)
2. [06-api-reference.md](06-api-reference.md)
3. [07-entity-model.md](07-entity-model.md)
4. [08-architecture-overview.md](08-architecture-overview.md)
5. [09-development-roadmap.md](09-development-roadmap.md)
6. [15-sign-conventions.md](15-sign-conventions.md)

For SmartBox-specific follow-up:

1. [02-smartbox-sbc.md](02-smartbox-sbc.md)
2. [14-smartbox-provisioning.md](14-smartbox-provisioning.md)
3. [15-sign-conventions.md](15-sign-conventions.md)

---

## Key Code Paths

### Cloud / service

- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/app/routes_pge_control.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/app/routes_pge_control.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/src/PgePilot/Api/TaskController.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/src/PgePilot/Api/TaskController.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/src/PgePilot/Api/ApiController.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/src/PgePilot/Api/ApiController.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/src/PgePilot/DataAdmin/DataAdminPgep.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/src/PgePilot/DataAdmin/DataAdminPgep.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/src/PgePilot/Services/SourcePolicyService.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/src/PgePilot/Services/SourcePolicyService.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/src/PgePilot/Services/ConnectorPowerTransformService.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/src/PgePilot/Services/ConnectorPowerTransformService.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/src/PgePilot/Services/PgeDataTimeSeriesStore.php`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/src/PgePilot/Services/PgeDataTimeSeriesStore.php)
- [`/Users/vladimiradam/projekty AI/pgepilot/pgepilot-service/migrations/009_history_lineage_and_policy_cutover.sql`](/Users/vladimiradam/projekty%20AI/pgepilot/pgepilot-service/migrations/009_history_lineage_and_policy_cutover.sql)

### SmartBox

- [`/Users/vladimiradam/projekty AI/pgepilot/sb/communication_controller/smartbox_service.py`](/Users/vladimiradam/projekty%20AI/pgepilot/sb/communication_controller/smartbox_service.py)
- [`/Users/vladimiradam/projekty AI/pgepilot/sb/rpc_client/models.py`](/Users/vladimiradam/projekty%20AI/pgepilot/sb/rpc_client/models.py)
- [`/Users/vladimiradam/projekty AI/pgepilot/sb/rpc_client/rpc_client.py`](/Users/vladimiradam/projekty%20AI/pgepilot/sb/rpc_client/rpc_client.py)
- [`/Users/vladimiradam/projekty AI/pgepilot/sb/rpc_client/main.py`](/Users/vladimiradam/projekty%20AI/pgepilot/sb/rpc_client/main.py)

---

## Production Checks

Runtime SHAs:

```bash
cat <<'EOF' | ssh root@pgepilot.cz 'bash -s'
for c in pgepilot_service pgepilot_worker1 pgepilot_worker2 pgepilot_worker3; do
  printf '%s ' "$c"
  docker exec "$c" sh -lc 'cd /var/www/html && git rev-parse HEAD'
done
docker exec pgepilot_servicedesk sh -lc 'cd /home/app2/pge-app && git rev-parse HEAD'
EOF
```

Task state:

```bash
ssh root@pgepilot.cz \
  "mysql -uroot -pVeslo123 -N -e \
  \"SELECT id, name, active, last_run
     FROM pgep_tasks.task_definitions
    WHERE id IN (18,19,20,21,22,23)
    ORDER BY id;\""
```

Recent failed backfill tasks:

```bash
ssh root@pgepilot.cz \
  "mysql -uroot -pVeslo123 -N -e \
  \"SELECT id, status, generated_at, completed_at,
           LEFT(response, 200), LEFT(param_data, 300)
     FROM pgep_tasks.tasks
    WHERE task_definition_id = 18
      AND status = 'failed'
    ORDER BY id DESC
    LIMIT 10;\""
```

Public web asset check:

```bash
ssh root@pgepilot.cz \
  \"curl -k -s https://app.pgepilot.cz \
    | grep -Eo 'index-[A-Za-z0-9_-]+\\.(js|css)' \
    | sort -u\"
```

Worker logs:

```bash
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_worker1 2>&1'
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_worker2 2>&1'
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_worker3 2>&1'
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_service 2>&1'
```

---

## Local Workspace Notes

Current local workspace is not perfectly clean:

- `pgepilot-service` has unrelated local changes in `infra/docker-compose.yml` and `infra/auth_srv/`
- do not assume local workspace equals production runtime
- before any new deploy, verify both local repo HEAD and live container HEAD
