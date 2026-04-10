# 10 -- Security

> Security audit findings, known risks, hardening plan, software versions, and RBAC status.
> Based on: `zadani/SECURITY_AUDIT_2026-04-03.md` (full 1025-line audit)
> Last updated: 2026-04-10

---

## Audit Summary (2026-04-03)

| Category | Critical | High | Medium | Low |
|----------|----------|------|--------|-----|
| Credentials & Secrets | 5 | 3 | 2 | 1 |
| Outdated software | 3 | 2 | 2 | 0 |
| Network security | 2 | 2 | 1 | 0 |
| Authentication & authorization | 2 | 3 | 2 | 0 |
| Configuration | 1 | 2 | 3 | 1 |
| **TOTAL** | **13** | **12** | **10** | **2** |

Some findings below are baseline audit items from 2026-04-03 and now include mitigation notes where the runtime has already been cleaned up.

---

## Critical Findings (13)

| ID | Finding | Where | Impact |
|----|---------|-------|--------|
| CRIT-01 | **PHP 8.1 EOL** (31.12.2025) | service, worker1/2/3, beapp | No security patches. CVE-2024-8925 (RCE), CVE-2024-8926 (DoS). Upgrade to PHP 8.3+ |
| CRIT-02 | **Node.js v18 EOL** (30.4.2025) | sb-manager on VPS | No security patches. Upgrade to Node.js 22 LTS |
| CRIT-03 | **Node.js v12** (EOL April 2022) | pgepilot_srv (legacy) | Hundreds of unpatched CVEs. Disable or rebuild |
| CRIT-04 | **JWT secrets = defaults** | auth_srv (auth.pgepilot.cz) | "secret" / "refreshSecret" as signing keys. Token forgery possible. Full RBAC bypass |
| CRIT-05 | **DB exposed to internet** | db.pgepilot.cz:3306 | NPM proxy routes to MariaDB. Full DB access with known password |
| CRIT-06 | **SSH keys in OneDrive** | pristupy a servery/ | Shared storage = anyone with OneDrive access can SSH to production |
| CRIT-07 | **Plaintext passwords in docs** | Multiple files | DB root, API keys, admin accounts all in cleartext Markdown |
| CRIT-08 | **Weak shared passwords** | All servers | Same password (`[REDACTED]`) on 3+ systems. SmartBox defaults: admin1234, tech1234 |
| CRIT-09 | **DB root password hardcoded** | Source code, connection strings | Root password in PHP source files |
| CRIT-10 | **SMTP password in code** | EmailSender.php | Password in source code |
| CRIT-11 | **No Docker volume mounts** | pgepilot.cz containers | Data loss on `docker rm`. Only host MariaDB persists |
| CRIT-12 | **Incomplete backup** | pgepilot.cz | Only `pgepilot` DB backed up. Missing: pge_control, pge_data, pgepilot_data, pgep_tasks (total ~6 GB unprotected) |
| CRIT-13 | **No secrets management** | All servers | Passwords in .env files, no rotation, no encryption |

---

## High Findings (12)

| ID | Finding | Where | Impact |
|----|---------|-------|--------|
| HIGH-01 | **Node.js v20 approaching EOL** (April 2026) + active CVEs | jobmanager, SB1 | CVE-2025-59465 (HTTP/2 crash), CVE-2026-21636 (permission bypass). Patch to 20.18.2+ |
| HIGH-02 | **mysql2 npm CVE** | All Node.js apps | CVE-2024-21511: Arbitrary Code Injection via timezone (< 3.9.7) |
| HIGH-03 | **jsonwebtoken npm CVE** | All JWT apps | CVE-2022-23529: RCE (CVSS 7.6) in <= 8.5.1. Update to 9.0.0+ |
| HIGH-04 | **Wildcard CORS on SB Config API** | SmartBox :3000 | `Access-Control-Allow-Origin: *` allows any website to call API |
| HIGH-05 | **Nginx Proxy Manager CVEs** | pgepilot.cz | OS Command Injection via htpasswd. Update to latest |
| HIGH-06 | **Modbus TCP unencrypted** | SB1 → inverter | No auth, no encryption on Modbus. MITM can modify inverter commands |
| HIGH-07 | **Expiring SSL certs** | VPS wildcard | *.ra-energity.cz expires 2026-06-06 (63 days) |
| HIGH-08 | SB services on 0.0.0.0 | SB1 :3007, :3001 | health + rpc-client accessible from any interface |
| HIGH-09 | SB web UI passwords hardcoded | SB1 app.py | admin1234, tech1234, user1234 in source |
| HIGH-10 | Worker redeploy depends on host-side SSH key injection or manual `docker cp` | pgepilot.cz | Runtime can be kept in sync, but the container image does not carry persistent GitHub credentials, so redeploy is still operationally fragile |
| HIGH-11 | sb-manager instability history | VPS | 156 historical restarts recorded in PM2. Current uptime is 10 days, but root cause remains undocumented |
| HIGH-12 | Worker runtime fork not in git (mitigated 2026-04-10) | worker1/2/3 | Workers were backed up and reset to clean `main@b578bd8`; keep audit guardrails in place so drift does not return |

---

## Medium Findings (10)

| ID | Finding | Where |
|----|---------|-------|
| MED-01 | No application-level rate limiting on API | All API endpoints |
| MED-02 | No Content Security Policy (CSP) headers | All frontends |
| MED-03 | MariaDB 11 CVEs (CVE-2025-13699) | pgeweb |
| MED-04 | No DB user separation (everything uses root) | All databases |
| MED-05 | Inter-service HTTP (not HTTPS) | Docker containers |
| MED-06 | PasswordAuthentication yes on SSH | pgepilot.cz |
| MED-07 | Dead nginx proxy entries | sicak, calc, nab, taskmanager |
| MED-08 | plant_state_history ~5 GB | pgepilot.cz |
| MED-09 | Tailwind CSS from CDN | PGE App |
| MED-10 | SB1 test services in failing loop | sb-test-3000, sb-test-3001 |

---

## Software Version Matrix

| Component | Version | Status | Action |
|-----------|---------|--------|--------|
| PHP (service, workers) | 8.1.2 | **EOL** | Upgrade to 8.4 |
| Node.js (sb-manager) | 18.19.1 | **EOL** | Upgrade to 22 LTS |
| Node.js (jobmanager, SB1) | 20.x | EOL April 2026 | Patch to 20.18.2+, plan v22 |
| Node.js (pgepilot_srv) | 12.x | **EOL 2022** | Disable |
| MariaDB (pgepilot.cz) | 11.x | Supported | Update to 11.4.9+ |
| Python (SB1) | 3.13.5 | Current | OK |

---

## Hardening Plan (Prioritized)

### Week 1 -- Immediate

1. ~~Change DB password~~ → generate random, store in env var
2. ~~Block db.pgepilot.cz~~ → remove NPM proxy entry for :3306
3. ~~Generate random JWT secrets~~ → update auth_srv, rotate tokens
4. ~~Move SMTP password to env var~~ → update EmailSender.php
5. Upgrade PHP 8.1 → 8.4 in Docker images
6. Update Node.js sb-manager 18 → 22

### Week 2 -- Short Term

7. Expand backup → add pge_control, pge_data, pgepilot_data, pgep_tasks
8. Stop SB1 test services → `systemctl disable sb-test-3000 sb-test-3001`
9. Fix SB bind → 0.0.0.0 → 127.0.0.1 for health and rpc-client
10. Remove dead proxy entries → clean NPM config
11. Move SB passwords → from app.py to /etc/energity/credentials.yaml
12. Fix wildcard CORS on SB Config API

### Week 3-4 -- Medium Term

13. Docker volume mounts → persist container data
14. Auth middleware → enable on all write endpoints
15. Diagnose sb-manager → root cause of 156 historical restarts
16. npm install Tailwind → remove CDN dependency
17. Disable SSH PasswordAuthentication
18. Create dedicated DB users (least privilege)
19. Add rate limiting to API endpoints
20. Set up cron monitoring → alert on sync failure

### Ongoing

21. SSL cert monitoring (VPS wildcard expires 2026-06-06)
22. npm audit on all Node.js projects
23. Secrets manager evaluation (Doppler / Vault)

---

## RBAC Status

Implemented in PGE App:

| Role | Access |
|------|--------|
| Admin | All current pages including `/uzivatele` and `/alerty`, full CRUD where implemented |
| Operator | /alerty + standard views |
| User | Dashboard, CP detail, charts, domains, instalace, nastaveni |

API:
- `GET /api/v2/users/{id}/grants`
- `POST /api/v2/users/{id}/grants`
- `DELETE /api/v2/users/{id}/grants`

Grants link users to specific Collection Points and Control Domains.

---

## Firewall Status (verified 2026-04-09)

### pgepilot.cz
- **iptables DOCKER-USER**: SSH (22xx) whitelisted for 5.252.43.55, 46.135.6.209. Port 3306 DROP from non-Docker. Port 5000 logged + allowed from pgeusers (188.245.255.117)
- **fail2ban**: Active on port 22, autoban after 3 attempts
- **RISK**: db.pgepilot.cz still routes to MariaDB (NPM entry)

### ra-energity.cz
- **UFW**: ALLOW 22, 80, 443, 3055, 5001; LIMIT 20002, 20012
- **SSL**: Let's Encrypt wildcard *.ra-energity.cz (expires **2026-06-06**)
- **RISK**: sb-manazer.ra-energity.cz has no authentication

> Full audit: `zadani/SECURITY_AUDIT_2026-04-03.md` (private OneDrive)
