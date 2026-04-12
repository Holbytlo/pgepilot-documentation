# 05 -- Infrastructure

> Servers, Docker, databases, networking, deploy procedures, and backups.
> Last updated: 2026-04-12

> **Credentials note**: All passwords and keys are `[REDACTED]`.
> Actual values: `zadani/pristupy a servery/PGEERP_Knowledge_Base.md` (private OneDrive).

---

## Server Overview

| # | Server | IP | OS | Purpose | Provider |
|---|--------|----|----|---------|----------|
| 1 | pgepilot.cz | 88.99.187.9 | Ubuntu 22.04 ARM64 | PgePilot cloud (7 Docker containers) | Hetzner |
| 2 | ra-energity.cz | 195.201.19.103 | Ubuntu 24.04 x86_64 | SmartBox VPS (sb-manager, SSH tunnels) | Hetzner |
| 3 | pgeusers | 188.245.255.117 | -- | ERP applications (Pricelist, PMO, Auth, Admin, TaskHub) | Hetzner |
| 4 | pgen-data-analyse | 94.130.78.0 | -- | EnerSim simulation engine | Hetzner |

Servers 3 and 4 are part of the PGE ERP ecosystem (separate documentation).

---

## Server 1: pgepilot.cz

| Property | Value |
|----------|-------|
| RAM | 3.7 GB |
| Root disk | 38 GB SSD (26 GB free) |
| Data volume | /mnt/HC_Volume_101857288, 300 GB (205 GB free) |
| SSH | `ssh root@pgepilot.cz` (ed25519 key) |
| Firewall | iptables: SSH to containers whitelisted for specific IPs, fail2ban on port 22 |

### Docker Containers (8 running, verified 2026-04-10)

| Container | Ports (host) | SSH Port | Tech | Purpose |
|-----------|-------------|----------|------|---------|
| pgepilot_service | 8400 | 2214 | PHP 8.1, Slim4 | API backend, TaskManager |
| pgepilot_worker1 | 6001 | 2261 | PHP 8.1 | Task execution |
| pgepilot_worker2 | 6002 | 2262 | PHP 8.1 | Task execution (added) |
| pgepilot_worker3 | 6003 | 2263 | PHP 8.1 | Task execution (added) |
| pgepilot_jobmanager | 5000-5001 | 2205 | Node.js v20, PM2 | Job orchestrator + SB sim |
| pgepilot_servicedesk | 3050, 3060 | 2206 | Vue3, Node.js | Frontend + PGE App |
| pgepilot_auth_srv | 4000 | -- | Node.js, JWT | Authentication |
| nginx-proxy-manager | 80, 81, 443 | -- | nginx | Reverse proxy, SSL |

> `pgepilot_beapp` (legacy) exists as image but is NOT running.

### Container Access

```bash
# Standard containers:
ssh root@pgepilot.cz "docker exec -it pgepilot_service bash"
ssh root@pgepilot.cz "docker exec -it pgepilot_worker1 bash"
ssh root@pgepilot.cz "docker exec -it pgepilot_jobmanager sh"

# Servicedesk / frontend container:
ssh root@pgepilot.cz "docker exec -it pgepilot_servicedesk sh"
```

### Runtime Code State (verified 2026-04-12)

| Runtime path | Git state | Notes |
|--------------|-----------|-------|
| `pgepilot_service:/var/www/html` | clean `main@a921042` | Service and workers are uniform again after the history/auth follow-up work |
| `pgepilot_worker1:/var/www/html` | clean `main@a921042` | Uniform with `pgepilot_service` as of 2026-04-12 evening |
| `pgepilot_worker2:/var/www/html` | clean `main@a921042` | Uniform with `pgepilot_service` as of 2026-04-12 evening |
| `pgepilot_worker3:/var/www/html` | clean `main@a921042` | Uniform with `pgepilot_service` as of 2026-04-12 evening |
| `pgepilot_jobmanager:/home/app` | clean `main@4346047` | Runtime cleanup removed untracked `jobmanager.js.bak` / `jobmanager.js.pre_cleanup`; tracked `jobmanager.jszaloha` remains in repo |
| `pgepilot_servicedesk:/home/app2/pge-app` | clean `main@3d7e6bb` | Live frontend source tree now matches GitHub `main`; served bundle currently `/assets/index-lGjKNcFm.js` |
| `pgepilot_servicedesk:/home/app2/servicedesk` | clean `main@1c21bd4` | Admin UI checkout is clean |
| `pgepilot_auth_srv:/app` | no visible git checkout | Runtime is a deploy artifact, not a git-auditable repo |

### Nginx Proxy Manager -- Domain Routing

| Domain | Port | Backend |
|--------|------|---------|
| api.pgepilot.cz | :8400 | pgepilot_service |
| worker.pgepilot.cz | :6001 | pgepilot_worker1 |
| jobmanager.pgepilot.cz | :5000 | pgepilot_jobmanager |
| simsb.pgepilot.cz | :5001 | pgepilot_jobmanager (SB sim) |
| sd.pgepilot.cz | :3050 | pgepilot_servicedesk |
| app.pgepilot.cz | :3060 | PGE App (socat proxy) |
| auth.pgepilot.cz | :4000 | pgepilot_auth_srv |
| db.pgepilot.cz | :3306 | MariaDB -- **RISK: exposed to internet** |
| pgepilot.cz | :8000 | beapp (legacy, ignore) |

NPM Admin panel: `http://pgepilot.cz:81`

**Dead proxy entries** (should be removed): sicak, calc, nab, taskmanager -- point to dead ports.

### Firewall (iptables)

- **DOCKER-USER chain**: SSH ports (22xx) to containers allowed only from whitelisted IPs, all others DROP
- **Whitelisted IPs**: 5.252.43.55, 46.135.6.209
- **fail2ban-sshd**: Host SSH (port 22), autoban after 3 failed attempts

---

## Server 2: ra-energity.cz

See [02-smartbox-sbc.md](02-smartbox-sbc.md) for full details.

| Property | Value |
|----------|-------|
| RAM | 3.7 GB |
| Disk | 38 GB SSD (28 GB free) |
| Firewall (UFW) | ALLOW: 22, 80, 443, 3055, 5001. LIMIT: 20002, 20012 |
| SSL | Let's Encrypt wildcard *.ra-energity.cz (expires 2026-06-06) |
| SSH keepalive | `ClientAliveInterval 30`, `ClientAliveCountMax 3` (added 2026-04-12) |

**SSH tunnel keepalive (2026-04-12):** `/etc/ssh/sshd_config` now detects dead reverse tunnels within 90s. Previously, rebooted SB devices left zombie tunnel sessions blocking port re-binding for hours. Backup: `/etc/ssh/sshd_config.bak-20260412`.

---

## Databases (on pgepilot.cz host)

MariaDB running on host (not in Docker). Connection: `pgepilot.cz:3306`, user: `root`, password: `[REDACTED]`.

| Database | Purpose | Key Tables | Size (verified 2026-04-04) |
|----------|---------|------------|---------------------------|
| **pgepilot** | Legacy plants, machines, config | plants(36), machines(39), relay_groups, event_log, plant_realtime_status | **7166 MB** |
| **pgep_tasks** | Task management (shared) | task_definitions, tasks, tasks_archive | **4726 MB** |
| **pgepilot_dashboard** | Notifications | notification_outbox | **2549 MB** |
| **pgepilot_data** | Legacy time series (migrated from pgedata.cz 2026-03-28) | {code}_power_5m, {code}_energy_5m (55 tables) | **732 MB** |
| **pge_data** | New time series + forecast | canonical history (`power_1m`, `power_rt`, `energy_15m`) plus reporting profiles (`power_bf`, `power_1h`, `power_1d`) resolved from canonical history when no dedicated table exists, forecast tables (81 tables) | **480 MB** |
| **pge_control** | New entity model (27 tables) | cp_collection_points(35), cp_devices(26), cp_machines(40), realtime_state(23), cp_users(32) | **90 MB** |
| **pgep_cache** | Cache | -- | <1 MB |
| **pgep_users** | Users | -- | <1 MB |

### Undocumented / Legacy Databases

> **Action needed**: Ask whether these can be removed. Do NOT delete without confirmation.

| Database | Purpose | Size |
|----------|---------|------|
| pge_calc | Unknown (legacy?) | <1 MB |
| pgep_OPS1 | Unknown (legacy?) | <1 MB |
| pgep_udb_0 | Unknown (legacy?) | <1 MB |
| main | Unknown | <1 MB |
| pgepilot_bkp_20241005 | Backup from Oct 2024 | <1 MB |
| backup_pgep_va_20240921_154004 | VA backup from Sep 2024 | <1 MB |

### Backup

```bash
# Current backup cron (on host):
15 2 * * * /usr/local/bin/mysql-backup.sh
```

**Backs up**: `pgepilot` only.
**Missing from backup**: pge_control, pge_data, pgepilot_data, pgep_tasks, pgepilot_dashboard.

### Data Retention

```bash
# api_usage cleanup (added 2026-04-04):
25 3 * * * -- DELETE rows older than 30 days
```

`plant_state_history` (~5 GB) needs archival strategy.

---

## Network Diagram

```
                    INTERNET
                       |
          +------------+------------+
          |                         |
   pgepilot.cz:443            ra-energity.cz:443
   (NPM reverse proxy)        (nginx)
          |                         |
   +------+-------+          +-----+------+
   |      |       |          |     |      |
  :8400  :5000  :3050      :3055  :9100  :5001
  service jobmgr sdesk     sb-mgr router  sim
   |      |                        |
   |      +--every 3s--+           |
   |                   |    +------+------+
   +--POST /task--+    |    | SB1         |
                  |    |    | :80  Web UI |
              :6001    |    | :5010 DevCtrl|
              worker1  |    | :5011 SensDB |
                  |    |    | :3001 RPC   |
                  +--API+   +-------------+
                  |
          GoodWe/SolaX cloud API
```

---

## Deploy Procedures

### Service Container (git sync)
```bash
ssh root@pgepilot.cz "docker cp /root/.ssh/githolbytlo pgepilot_service:/tmp/githolbytlo && \
docker exec pgepilot_service sh -lc 'chmod 600 /tmp/githolbytlo && cd /var/www/html && \
GIT_SSH_COMMAND=\"ssh -i /tmp/githolbytlo -o IdentitiesOnly=yes\" git fetch origin main && \
git checkout -B main origin/main && git reset --hard origin/main && rm -f /tmp/githolbytlo'"
```

### Worker Containers (git sync from host)

Workers are currently clean on `main@a921042`. The practical limitation is credentialing: containers do not keep the GitHub SSH key permanently, so host-side sync temporarily injects the key, fetches, resets, then deletes the key again.

```bash
# Clean sync worker1/2/3 from GitHub main
ssh root@pgepilot.cz "
  for c in pgepilot_worker1 pgepilot_worker2 pgepilot_worker3; do
    docker cp /root/.ssh/githolbytlo \$c:/tmp/githolbytlo
    docker exec \$c sh -lc '
      chmod 600 /tmp/githolbytlo &&
      cd /var/www/html &&
      GIT_SSH_COMMAND=\"ssh -i /tmp/githolbytlo -o IdentitiesOnly=yes\" git fetch origin main &&
      git checkout -B main origin/main &&
      git reset --hard origin/main &&
      rm -f /tmp/githolbytlo
    '
  done
"
```

`docker cp` is still possible for emergency one-file hotfixes, but after any live patch the matching git commit should land in GitHub and the runtime should be reset back to a clean checkout.

### Servicedesk/PGE App (docker cp + nsenter build)

`docker exec` with `sh` works on the current container. `nsenter` is optional, not required for a normal build.
Current production checkout is clean `main@3d7e6bb` and serves `/assets/index-lGjKNcFm.js`.

```bash
# 1. Copy file to container
docker cp /tmp/Component.vue pgepilot_servicedesk:/home/app2/pge-app/src/views/Component.vue

# 2. Build inside container
docker exec pgepilot_servicedesk sh -lc 'cd /home/app2/pge-app && npm run build'

# 3. Verify
curl -s -o /dev/null -w '%{http_code}' http://localhost:3060
```

### SmartBox (preferred rsync deploy; some live devices still have git checkout)

Recommended deploy method is rsync from Mac. Live SB1 currently still has `git` installed and `/opt/energity/sb/.git`, but device-side git is not the target operational model:

```bash
# 1. Backup on device
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 \
  "tar czf /tmp/sb-backup-\$(date +%Y%m%d-%H%M%S).tar.gz -C /opt/energity sb"

# 2. Stop services
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 \
  "systemctl stop energity-device-controller energity-comm-controller \
   energity-rpc-client energity-config-api energity-web-interface \
   energity-local-db energity-logs-db"

# 3. Rsync from Mac (excludes preserve per-machine configs and data)
rsync -az --delete \
  -e 'ssh -J limited@ra-energity.cz -p <SSH_PORT>' \
  --exclude='.git/' --exclude='.venv/' --exclude='__pycache__/' \
  --exclude='*.pyc' --exclude='.DS_Store' --exclude='._*' \
  --exclude='local_database/*.db*' --exclude='local_database/*.sqlite*' \
  --exclude='device_controller/devices_config.yaml' \
  --exclude='communication_controller/comm_controller_config.yaml' \
  --exclude='communication_controller/smartbox_config.yaml' \
  --exclude='rpc_client/rpc_client_config.yaml' \
  --exclude='web_interface/config/backups/' \
  --exclude='logs/' --exclude='data/' \
  ~/projekty\ AI/pgepilot/sb/ root@127.0.0.1:/opt/energity/sb/

# 4. Fix ownership and start
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 "
  chown -R energity:energity /opt/energity/sb/
  chown -R root:root /opt/energity/sb/.venv/
  systemctl reset-failed 'energity-*'
  systemctl start energity-local-db energity-logs-db
  sleep 2
  systemctl start energity-device-controller energity-comm-controller \
    energity-config-api energity-web-interface energity-rpc-client"

# 5. Verify
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 \
  "systemctl list-units 'energity-*' --no-pager --no-legend | awk '{print \$1, \$3, \$4}'"
```

SSH ports: sb1=20002, sb4=20032, sb7=20062.

SmartBox auth rollout status (verified 2026-04-12):
- migrated to `service.pgepilot.cz`: SB1, SB4, SB7
- still on legacy `auth.pgepilot.cz`: remaining boxes
- legacy `pgepilot_auth_srv` remains active intentionally during gradual migration

**Important excludes rationale:**
- `devices_config.yaml` -- per-machine device config (GoodWe IP, Deye IP, enabled state)
- `rpc_client_config.yaml` -- per-machine cloud credentials (sbx_xxx)
- `smartbox_config.yaml` -- per-machine SmartBox identity (`smartbox_id`, `collection_point_id`, `device_id`, `machine_id`)
- `local_database/*.db*` -- historical sensor/log data
- `.venv/` -- stays on device, no new deps needed unless requirements.txt changes

### Post-Deploy Verification

After any deploy, verify:
```bash
# Check containers are running
docker ps --format '{{.Names}}: {{.Status}}'

# Check PM2 processes
docker exec pgepilot_jobmanager pm2 list
docker exec pgepilot_servicedesk pm2 list

# Check API health
curl -s http://localhost:8400/api/v2/health

# Check recent tasks
mysql -u root -p[REDACTED] pgep_tasks -e "SELECT id, name, active FROM task_definitions WHERE active=1;"
```

---

## Scheduled Jobs Map (verified 2026-04-04)

All scheduled execution across the entire PgePilot ecosystem in one place.

### Host Crontab (pgepilot.cz)

| Schedule | Script | Purpose |
|----------|--------|---------|
| `15 2 * * *` | mysql-backup.sh | Backup (only pgepilot DB!) |
| `*/5 * * * *` | docker_healthcheck.sh | Docker container health monitoring |
| `25 3 * * *` | api_usage cleanup | DELETE api_usage rows > 30 days |

### Service Container Crontab

| Schedule | Script | Purpose |
|----------|--------|---------|
| `3 0 1 4 *` | enable_solax.php | One-time SolaX API re-enable (April 2026) |

> **Note**: Sync crons (sync_realtime, sync_history, etc.) have been migrated OUT of crontab into task_definitions 20-22.

### JobManager / Scheduler Reality

As of 2026-04-07, current production recurring execution is DB-driven:

```text
task_definitions (MariaDB) -> service /run-tasks -> JobManager /add_task -> worker /task
```

The older `Holbytlo/pgepilot-js` `/scheduled_tasks` and `/run_scheduled_task` handlers remain commented on current `main`. They should be treated as legacy notes, not as the primary production scheduler model.

### Task System (JobManager → Service → Workers, every 3s)

| ID | Name | Active | Interval |
|----|------|--------|----------|
| 15 | Plant Health Check | Yes | Repeating |
| 17 | Control Relays | Yes | Repeating |
| 18 | Historical Data Backfill | Yes | Repeating |
| 19 | Energy Data Backfill (GoodWe) | Yes | Repeating |
| 20 | Sync Realtime | **No** | Migrated from cron, still disabled |
| 21 | Sync History | **No** | Migrated from cron, still disabled |
| 22 | Record Realtime to History | **Yes** | Reactivated 2026-04-12; records `realtime_state` into canonical `power_1m` every minute |
| 23 | Compute Energy 15m | Yes | `every:900seconds` |
| 24 | PV Forecast (forecast.solar) | **No** | `every:3600seconds` |
| 25 | Load Forecast | **No** | `every:3600seconds` |
| 26 | PV Forecast OpenMeteo | Yes | `every:3600seconds` |
| 27 | Forecast Correction | Yes | `daily:01:05` |
| 28 | Task Cleanup | Yes | `every:3600seconds` |
| 29 | Collect Realtime Data | Yes | `every:300seconds` |
| 30 | OTE Spot Import Today | Yes | `daily:00:05` |
| 31 | OTE Spot Import Tomorrow | Yes | `daily:12:10` |

### OTE Import Verification

Production verification on 2026-04-07 confirmed:

- task definition `30` completed through JobManager/worker flow and imported 96 PT15M rows for today
- task definition `31` completed through JobManager/worker flow and imported 192 PT15M rows for today + tomorrow
- the importer route is `GET /ote/import` in `pgepilot_service`, while workers call it through `TaskController::runOteSpotImport()`

---

## API Connectors (External)

| Connector | Base URL | Auth | Rate Limit |
|-----------|----------|------|------------|
| GoodWe OpenAPI | eu.openapi.semsportal.com | MD5 hash -> 2h token | 3600/hour |
| GoodWe Web API | eu.semsportal.com/api | CrossLogin -> base64 token | -- |
| SolaxCloud | global.solaxcloud.com/api/v2 | tokenId header | 10,000/day, 100/min |
| forecast.solar | forecast.solar/api | API key | Professional tier |
| Open-Meteo | api.open-meteo.com | None (free) | -- |

Credentials: `[REDACTED]` -- see private OneDrive files.
