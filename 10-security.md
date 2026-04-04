# 10 -- Security

> Known security risks, hardening plan, and RBAC status.
> Last updated: 2026-04-04

---

## Critical Risks

These should be addressed immediately.

| # | Risk | Impact | Current State |
|---|------|--------|---------------|
| 1 | **DB password hardcoded** | Full database access if code leaks | Root password `[REDACTED]` is in source code and connection strings |
| 2 | **db.pgepilot.cz exposes port 3306 to internet** | Anyone can attempt DB login | NPM proxy entry routes to MariaDB |
| 3 | **JWT secrets are defaults** | Token forgery possible | auth_srv uses `"secret"` / `"refreshSecret"` as signing keys |
| 4 | **SMTP password hardcoded** | Email impersonation | Password in `EmailSender.php` source code |
| 5 | **No Docker volume mounts** | Data loss on `docker rm` | Container data is ephemeral; only host MariaDB persists |
| 6 | **Incomplete backup** | Data loss | Backup only covers `pgepilot` DB, missing: pge_control, pge_data, pgepilot_data, pgep_tasks |

---

## Medium Risks

| # | Risk | Impact | Current State |
|---|------|--------|---------------|
| 7 | SmartBox services on 0.0.0.0 | Unauthorized access to SB health/RPC | energity-health (:3007), energity-rpc-client (:3001) bind to all interfaces |
| 8 | SB web UI passwords hardcoded | Unauthorized SB access | Credentials in `app.py`, plan: move to `/etc/energity/credentials.yaml` |
| 9 | Worker deploy via docker cp | No version control on worker | Worker has no functional git, files copied from service container |
| 10 | Servicedesk docker exec broken | Complicated deploy process | OCI error, must use nsenter |
| 11 | sb-manager instability | Management disruption | 155 restarts in 4 days |
| 12 | SB1 failing test services | Resource waste, confusion | sb-test-3000/3001 in failing loop |
| 13 | Tailwind CSS from CDN | Production dependency on CDN | Should be npm installed |
| 14 | Dead nginx proxy entries | Attack surface | sicak, calc, nab, taskmanager point to dead ports |
| 15 | plant_state_history ~5 GB | Disk pressure | Needs archival strategy |
| 16 | PasswordAuthentication yes | Brute force risk on SSH | pgepilot.cz allows password auth (mitigated by fail2ban) |
| 17 | GoodWe cache methods not in git | Code drift | 4 new cache methods on worker1 are local edits only |

---

## Hardening Plan (Prioritized)

### Immediate (Week 1)

1. **Change DB password** -- generate random password, store in env var, update all connection strings
2. **Block db.pgepilot.cz** -- remove NPM proxy entry for port 3306
3. **Generate random JWT secrets** -- update auth_srv config, rotate all tokens
4. **Move SMTP password to env var** -- update EmailSender.php

### Short Term (Week 2)

5. **Expand backup script** -- add pge_control, pge_data, pgepilot_data, pgep_tasks
6. **Stop SB1 test services** -- `systemctl disable sb-test-3000 sb-test-3001`
7. **Fix SB bind addresses** -- change 0.0.0.0 to 127.0.0.1 for health and rpc-client
8. **Remove dead proxy entries** -- clean up NPM config
9. **Move SB web passwords** -- from app.py to /etc/energity/credentials.yaml

### Medium Term (Week 3-4)

10. **Docker volume mounts** -- ensure container data persists across restarts
11. **Auth middleware on all write endpoints** -- currently some endpoints lack auth checks
12. **Diagnose sb-manager** -- root cause of 155 restarts
13. **npm install Tailwind** -- remove CDN dependency
14. **Disable SSH password auth** -- set PasswordAuthentication to no
15. **Cron monitoring** -- alert when sync crons fail

---

## RBAC Status

RBAC (Role-Based Access Control) is implemented in the PGE App:

- **Admin**: Full access to all pages including /predikce, /uzivatele, /alerty
- **Operator**: Access to /alerty + standard views
- **User**: Dashboard, CP detail, charts, domains

User grants are managed via:
- `GET /api/v2/users/{id}/grants`
- `POST /api/v2/users/{id}/grants`
- `DELETE /api/v2/users/{id}/grants`

Grant types link users to specific Collection Points and Control Domains.

---

## Firewall Status

### pgepilot.cz
- **iptables DOCKER-USER chain**: SSH to containers (22xx ports) whitelisted for 2 IPs
- **fail2ban**: Active on port 22, autoban after 3 attempts
- **Risk**: db.pgepilot.cz routes to MariaDB port 3306

### ra-energity.cz
- **UFW**: ALLOW 22, 80, 443, 3055, 5001; LIMIT 20002, 20012
- **SSL**: Let's Encrypt wildcard (expires 2026-06-06)
- **Risk**: sb-manazer.ra-energity.cz has no authentication
