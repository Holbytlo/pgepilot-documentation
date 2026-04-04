# 05 -- Infrastructure

> Servers, Docker, databases, networking, deploy procedures, and backups.
> Last updated: 2026-04-04

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

### Docker Containers (7)

| Container | Ports (host:container) | SSH Port | Tech | Purpose |
|-----------|----------------------|----------|------|---------|
| pgepilot_service | 8400->80 | 2214 | PHP 8.1, Slim4 | API backend, TaskManager |
| pgepilot_worker1 | 6001->80 | 2261 | PHP 8.1 | Task execution |
| pgepilot_jobmanager | 5000, 5001 | 2205 | Node.js v20, PM2 | Job orchestrator + SB sim |
| pgepilot_servicedesk | 3050, 3060 | 2206 | Vue3, Node.js | Frontend + PGE App |
| pgepilot_auth_srv | 4000 | -- | Node.js, JWT | Authentication |
| pgepilot_beapp | 8001->80 | 2201 | PHP 8.1 | LEGACY (do not use) |
| nginx-proxy-manager | 80, 81, 443 | -- | nginx | Reverse proxy, SSL |

### Container Access

```bash
# Standard containers:
ssh root@pgepilot.cz "docker exec -it pgepilot_service bash"
ssh root@pgepilot.cz "docker exec -it pgepilot_worker1 bash"
ssh root@pgepilot.cz "docker exec -it pgepilot_jobmanager bash"

# Servicedesk (docker exec fails with OCI error -- use nsenter):
SD_PID=$(ssh root@pgepilot.cz "docker inspect pgepilot_servicedesk --format '{{.State.Pid}}'")
ssh root@pgepilot.cz "nsenter -t $SD_PID -m -u -i -n -p -- bash"
```

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

---

## Databases (on pgepilot.cz host)

MariaDB running on host (not in Docker). Connection: `pgepilot.cz:3306`, user: `root`, password: `[REDACTED]`.

| Database | Purpose | Key Tables | Size |
|----------|---------|------------|------|
| **pgepilot** | Legacy plants, machines, config | plants(36), machines(39), relay_groups, event_log, plant_realtime_status | ~6 GB (plant_state_history ~5 GB) |
| **pge_control** | New entity model | cp_collection_points(23), cp_devices(26), cp_machines(40), cp_commands, realtime_state(23), cp_users(19) | Small |
| **pge_data** | New time series + forecast | {code}_power_1m, {code}_energy_15m, pv_forecast, load_forecast, weather_forecast (81 tables) | Growing |
| **pgepilot_data** | Legacy time series (migrated from pgedata.cz 2026-03-28) | {code}_power_5m, {code}_energy_5m (55 tables) | Medium |
| **pgep_tasks** | Task management (shared) | task_definitions, tasks, tasks_archive (~1.6M rows) | Medium |
| **pgepilot_dashboard** | Notifications | notification_outbox | Small |
| **pgep_cache** | Cache | -- | Small |
| **pgep_users** | Users | -- | Small |

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

### Service Container (git pull)
```bash
ssh root@pgepilot.cz "docker exec pgepilot_service bash -c 'cd /var/www/html && git pull'"
```

### Worker Container (docker cp -- no functional git)
```bash
ssh root@pgepilot.cz "docker cp pgepilot_service:/var/www/html/src/File.php /tmp/f && \
  docker cp /tmp/f pgepilot_worker1:/var/www/html/src/File.php"
```

### Servicedesk/PGE App (docker cp + nsenter build)
```bash
docker cp /tmp/Component.vue pgepilot_servicedesk:/home/app2/pge-app/src/views/...
SD_PID=$(docker inspect pgepilot_servicedesk --format '{{.State.Pid}}')
nsenter -t $SD_PID -m -u -i -n -p -- bash -c 'cd /home/app2/pge-app && npm run build'
```

### SmartBox (git pull on SB1)
```bash
ssh -J limited@ra-energity.cz -p 20002 root@127.0.0.1
cd /opt/energity/sb && git pull
systemctl restart energity-device-controller
```

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
