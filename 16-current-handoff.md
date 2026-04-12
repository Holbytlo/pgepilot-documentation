# 16 -- Current Handoff

> Short operational handoff for the next chat/window.
> Last updated: 2026-04-12

---

## Live Snapshot

This file exists because the production state changed across multiple chats on the same day and the runtime is no longer fully uniform.

Current verified state:

- `pgepilot_service` runtime: `main@da784a6` (`fix: bind smartbox local auth middleware helper`)
- `pgepilot_worker1/2/3` runtime: `main@a04e0ba` (`Route history pipeline through canonical pge_data datasets`)
- `pgepilot_servicedesk:/home/app2/pge-app`: `main@3d7e6bb`
- public `app.pgepilot.cz` bundle: `index-lGjKNcFm.js` + `index-DfwOyOjc.css`
- `task_definitions`: `18=active`, `19=active`, `20=disabled`, `21=disabled`, `22=active`, `23=active`

Important:

- the history cutover and the SmartBox/auth follow-up were not the same deploy
- the service container is now one commit ahead of the workers
- do not assume `service` and `worker1/2/3` are on the same SHA

---

## What Is Already Done

- canonical history now targets `pge_data`
- GoodWe backfill writes `pge_data.{code}_power_1m`
- SmartBox `smartboxSendData` is mapped into canonical `power_1m` / reported `energy_15m`
- `power_bf` is treated as a reporting profile over canonical history
- `task 22` (`recordRealtimeToHistory`) was reactivated and writes current `realtime_state` into `power_1m`
- `task 23` computes `energy_15m` from `power_1m`
- web app already uses the new history `usage` contract

---

## What Is Still Open

### 1. Periodic Task 18 failures

`Historical Data Backfill` is still failing on some runs with HTTP `500`.

Verified examples:

- `2026-04-12 15:55:16` -> failed
- `2026-04-12 16:05:16` -> failed
- `2026-04-12 16:15:16` -> failed

One captured failed payload:

```json
{
  "controllerFunction": "backfillHistoricalData",
  "params": {
    "plantId": "df559764-d5f3-4101-a3cb-cc1b7d1831f3",
    "dateFrom": "",
    "dateTo": "",
    "maxDays": 7
  }
}
```

So the history model is deployed, but the GoodWe backfill is not yet fully clean.

### 2. Runtime split

`pgepilot_service` is on `da784a6`, while `worker1/2/3` remain on `a04e0ba`.

If the next task touches:

- SmartBox/local auth helper -> inspect `da784a6`
- worker-executed history/backfill code -> workers still run `a04e0ba`

### 3. Auth context

Auth was handled in a different chat.

For this handoff:

- do not treat auth as part of the history-cutover task
- do not silently change auth contracts while debugging history
- if a task is about SB payload consistency, stay in SB/service payload scope unless auth is explicitly the bug

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
ssh root@pgepilot.cz 'tail -n 80 /var/log/pgepilot/worker/taskcontroller.log'
ssh root@pgepilot.cz 'tail -n 80 /var/log/pgepilot/worker/taskcontroller.err'
```

---

## Local Workspace Notes

Current local workspace is not perfectly clean:

- `pgepilot-service` has unrelated local changes in `infra/docker-compose.yml` and `infra/auth_srv/`
- do not assume local workspace equals production runtime
- before any new deploy, verify both local repo HEAD and live container HEAD
