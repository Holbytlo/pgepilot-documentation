# Server Audit Procedure

> Step-by-step guide for verifying real server state against documentation.
> Use this when updating docs or onboarding to a new session.
> Last updated: 2026-04-10

---

## Prerequisites

```bash
# SSH key setup (from current workspace)
cp "secure-access/original-pristupy-a-servery/id_ed25519" /tmp/pge_ssh_key
chmod 600 /tmp/pge_ssh_key

# GitHub SSH key (for git push)
cp "secure-access/ssh/githolbytlo" /tmp/githolbytlo
chmod 600 /tmp/githolbytlo
```

---

## 1. pgepilot.cz (PgePilot Cloud)

### Connect
```bash
ssh -i /tmp/pge_ssh_key -o StrictHostKeyChecking=no root@pgepilot.cz
```

### System basics
```bash
uname -a                          # OS, kernel, architecture
df -h / /mnt/HC_Volume_101857288  # Disk usage (root + data volume)
free -h                           # RAM usage
```

### Docker containers
```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker images --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
```

Check: How many containers? Are worker2/worker3 present? Is beapp running?

### Databases
```bash
# List all databases
mysql -u root -p[REDACTED] -e 'SHOW DATABASES;'

# Database sizes (MB)
mysql -u root -p[REDACTED] -e '
  SELECT table_schema AS db,
         ROUND(SUM(data_length + index_length)/1024/1024, 1) AS size_mb
  FROM information_schema.tables
  GROUP BY table_schema ORDER BY size_mb DESC;'

# pge_control table counts
mysql -u root -p[REDACTED] -h 127.0.0.1 -e '
  SELECT TABLE_NAME, TABLE_ROWS
  FROM information_schema.tables
  WHERE table_schema="pge_control" ORDER BY TABLE_NAME;'

# Task definitions (pgep_tasks)
mysql -u root -p[REDACTED] -h 127.0.0.1 pgep_tasks -e '
  SELECT id, name, active FROM task_definitions ORDER BY id;'

# Active plants count
mysql -u root -p[REDACTED] -h 127.0.0.1 pge_control -e '
  SELECT COUNT(*) AS total,
         SUM(status="active") AS active
  FROM cp_collection_points;'

# Connector status
mysql -u root -p[REDACTED] -h 127.0.0.1 pge_control -e '
  SELECT type, COUNT(*) as cnt,
         SUM(realtime_enabled) as rt_on,
         SUM(backfill_enabled) as bf_on
  FROM connector_config GROUP BY type;'
```

### Container internals
```bash
# Service container: crontab + git
docker exec pgepilot_service crontab -l
docker exec pgepilot_service bash -lc 'cd /var/www/html && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -40'

# JobManager: runtime lives in /home/app
docker exec pgepilot_jobmanager sh -lc 'cd /home/app && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -40'

# Current production scheduler truth lives in DB task_definitions
mysql -u root -p[REDACTED] pgep_tasks -e "SELECT id, name, active, repeating_time FROM task_definitions ORDER BY id;"
mysql -u root -p[REDACTED] pgep_tasks -e "SELECT id, task_definition_id, status, completed_at FROM tasks ORDER BY id DESC LIMIT 20;"

# Servicedesk frontend/admin checkouts
docker exec pgepilot_servicedesk sh -lc 'cd /home/app2/pge-app && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -40'
docker exec pgepilot_servicedesk sh -lc 'cd /home/app2/servicedesk && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -40'

# Auth runtime (may be deploy artifact, not git)
docker exec pgepilot_auth_srv sh -lc 'pwd; find / -maxdepth 4 -name .git 2>/dev/null | head -10'

# Worker git status
docker exec pgepilot_worker1 bash -lc 'cd /var/www/html && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -60'
docker exec pgepilot_worker2 bash -lc 'cd /var/www/html && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -60'
docker exec pgepilot_worker3 bash -lc 'cd /var/www/html && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short | head -60'
```

### Host crontab
```bash
crontab -l
```

### Firewall
```bash
iptables -L DOCKER-USER -n | head -25
```

### Nginx Proxy Manager
```bash
# Admin panel: http://pgepilot.cz:81
# Check proxy hosts via SQLite (if NPM uses SQLite):
sqlite3 /opt/docker/nginx-proxy-manager/data/database.sqlite "SELECT domain_names, forward_host, forward_port FROM proxy_host WHERE enabled=1;" 2>/dev/null
# Or check via NPM API / admin panel
```

---

## 2. ra-energity.cz (SmartBox VPS)

### Connect
```bash
ssh -i /tmp/pge_ssh_key -o StrictHostKeyChecking=no limited@ra-energity.cz
```

### System basics
```bash
uname -a && df -h / && free -h
```

### Services
```bash
pm2 list                                    # PM2 processes (sb-manager, sim)
cd /opt/sb-manager && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short
cd /opt/sb-router && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short
systemctl status sb-router --no-pager       # SB router
systemctl status nginx --no-pager           # nginx
```

### SSH tunnels (which SmartBoxes are connected)
```bash
ss -tlnp | grep -E '200[0-9][0-9]'
# Port schema: base = 20000 + (N-1)*10
# SB1: 20000-20009, SB2: 20010-20019, etc.
# Offsets: +0=nginx, +2=ssh, +3=config, +4=api, +5=ops, +9=health
```

### Firewall + SSL
```bash
sudo ufw status
sudo certbot certificates | head -20    # SSL expiry check
```

### Nginx configs
```bash
ls /etc/nginx/conf.d/
cat /etc/sb-router/config.json
```

---

## 3. SmartBox SB1 (via VPS jump)

### Connect
```bash
# From VPS:
ssh -p 20002 root@127.0.0.1

# Or direct jump from local:
ssh -i /tmp/pge_ssh_key -o StrictHostKeyChecking=no -J limited@ra-energity.cz -p 20002 root@127.0.0.1
```

### System basics
```bash
uname -a && df -h / && free -h
```

### Services
```bash
# All energity services
systemctl list-units --type=service --all | grep -E 'energity|sb-|ra-|nginx'

# Ports in use
ss -tlnp | grep LISTEN

# Check for failing services
systemctl --failed
```

### Software
```bash
python3 --version && node --version

# Git status
cd /opt/energity/sb && git rev-parse --abbrev-ref HEAD && git rev-parse HEAD && git status --short

# Device configuration
cat /opt/energity/sb/device_controller/devices_config.yaml
```

### Quick health check
```bash
# Device controller
curl -s http://127.0.0.1:5010/status 2>/dev/null || echo "DC not responding"

# Sensor DB
curl -s http://127.0.0.1:5011/api/sensor-data/latest 2>/dev/null | head -5

# Health aggregator
curl -s http://127.0.0.1:3007/health 2>/dev/null

# RPC client
curl -s http://127.0.0.1:3001/status 2>/dev/null || echo "RPC not responding"
```

---

## 4. What to Compare

After gathering data, compare against documentation files:

| Check | Doc file | What to verify |
|-------|----------|---------------|
| Container count + names | 01-pgepilot-cloud.md, 05-infrastructure.md | docker ps vs documented containers |
| DB sizes + table counts | 05-infrastructure.md, 07-entity-model.md | Sizes, row counts, new tables |
| Task definitions | 01-pgepilot-cloud.md | Active tasks, new tasks, intervals |
| Collection point count | 07-entity-model.md, 09-development-roadmap.md | cp_collection_points count |
| Service crontab | 01-pgepilot-cloud.md | What's in crontab vs what docs say |
| VPS tunnel ports | 02-smartbox-sbc.md | Active SB connections |
| SB services | 02-smartbox-sbc.md | Running services, failing services |
| Git HEADs | All docs | Current commits vs documented |
| SSL expiry | 02-smartbox-sbc.md | Certificate expiration dates |

---

## 5. Quick One-Liner Audit (all 3 servers)

Run these in parallel for a fast status check:

```bash
# pgepilot.cz
ssh -i /tmp/pge_ssh_key root@pgepilot.cz \
  "docker ps --format '{{.Names}}:{{.Status}}' && echo '---' && \
   mysql -uroot -p[REDACTED] -h127.0.0.1 pge_control -N -e 'SELECT COUNT(*) FROM cp_collection_points' && \
   echo 'CPs' && crontab -l | wc -l && echo 'cron lines'"

# ra-energity.cz
ssh -i /tmp/pge_ssh_key limited@ra-energity.cz \
  "pm2 list --no-color && ss -tlnp | grep '200[0-9][0-9]' | wc -l && echo 'tunnel ports'"

# SB1 (via jump)
ssh -i /tmp/pge_ssh_key -J limited@ra-energity.cz -p 20002 root@127.0.0.1 \
  "systemctl --failed --no-pager && systemctl list-units --type=service --state=running | grep energity | wc -l && echo 'running services'"
```
