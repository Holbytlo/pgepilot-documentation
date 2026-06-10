# PgePilot prod security posture audit — 2026-06-11 (PGP-032)

Read-only audit živého `pgepilot.cz`. Provedl claude_valVA_v2. ŽÁDNÁ změna (firewall/cron/NPM/secrets/restart).
Metodika: externí TCP reachability z Macu + server-side read-only (`ss`, `iptables -L`, `docker ps`, čtení
backup skriptu a sshd_config). Hodnoty secrets se nečetly.

## Pořadí podle závažnosti

### 🔴 H1 — Container SSH porty otevřené celému internetu, root login, bez fail2ban
- `0.0.0.0:2205` (jobmanager), `2206` (servicedesk), `2214` (service), `2261/2262/2263` (worker1/2/3) → docker-proxy bind na všechny IP.
- **iptables `DOCKER-USER` tyto porty NEomezuje** — řídí jen `:22` (host SSH, jen 5.252.43.55 + 46.135.6.209), `:3306`, `:5000/:5001`. Porty 22xx propadnou na `RETURN` (accept) → dostupné z celého internetu (potvrzeno: připojení na `:2214` prošlo z nebílelistované IP).
- `pgepilot_service` i `pgepilot_jobmanager` mají `PermitRootLogin yes`; `PasswordAuthentication` není explicitně nastaveno → sshd default `yes`.
- `fail2ban` má jediný jail `sshd` na hostu (port 22) — container SSH porty **nekryje** = žádná brute-force ochrana.
- **Dokumentace lže:** `pristupy a servery` tvrdí „SSH na kontejnery (porty 22xx) povoleno jen z 5.252.43.55 a 46.135.6.209, ostatní DROP" — v běžícím iptables takové pravidlo NENÍ.
- Dopad: root SSH brute-force surface na 6 portech bez rate-limitu.
- Náprava: přidat do DOCKER-USER DROP pro 2205/2206/2214/2261-2263 mimo whitelist (jako u :22); v kontejnerech `PermitRootLogin no` + `PasswordAuthentication no` (jen klíče); rozšířit fail2ban na container porty NEBO je vůbec nepublikovat na 0.0.0.0 (bind jen na management IP / WireGuard).

### 🔴 H2 — :8400 (pgepilot_service API) veřejný → zesiluje PGP-058
- `0.0.0.0:8400->80` na `pgepilot_service` je přímo z internetu (potvrzeno z Macu). Spolu s PGP-058 (neautentizované debug/relé GET routy v Route_Pgep.php) to znamená, že `/test-rele` apod. jsou dosažitelné **přímo na `pgepilot.cz:8400`, mimo NPM proxy** — žádná proxy-auth je negatí.
- Dále veřejné: `:8401` (service_demo), `:6001-6003` (worker HTTP), `:4000` (pgepilot_auth_srv).
- Náprava: zvýšit prioritu PGP-058 (deploy hotfixu); API porty pustit jen přes NPM (bind 127.0.0.1 / docker-internal), ne 0.0.0.0.

### 🟠 M1 — mariadbd bind 0.0.0.0:3306 (broad), chráněno jen iptables
- `mariadbd` (host proces) poslouchá na `0.0.0.0:3306`. Externě NEdostupné — DOCKER-USER rules 10/11 DROP non-docker zdroje, ACCEPT jen z 188.245.255.117 + docker sítí (ověřeno: z Macu `:3306` filtered).
- Defense-in-depth gap: bind je široký; při flush/přeřazení iptables by byl DB okamžitě veřejný. Náprava: bind `127.0.0.1` + docker-internal, ne 0.0.0.0.

### 🟠 M2 — Backup pokrývá jen hardcoded seznam schémat
- `/usr/local/bin/mysql-backup.sh` iteruje pevné pole `SCHEMAS[@]` (ne `--all-databases`); zvlášť zálohuje mysql grants. Cíl `/mnt/HC_Volume_101857288/backups/mysql`. Nové schéma přidané do DB se **tiše nezálohuje**, dokud se nedoplní do pole.
- Náprava: buď `--all-databases`, nebo guard který porovná živá schémata vs `SCHEMAS` a alertuje na nepokrytá; ověřit restore (PGP-032 roadmap 3.x).

### ⚪ Nedokončeno read-only (potřebuje další přístup)
- Dead NPM proxy entries — vyžaduje NPM admin DB/API (`:81`).
- Hardcoded/default secrets — záměrně nečteno; doporučen samostatný řízený sken env/config.

## Souhrn portů (0.0.0.0 listen na hostu)
veřejné a bez restrikce: 80, 81, 443 (NPM), 25 (smtp), 2205, 2206, 2214, 2261-2263 (container SSH), 6001-6003, 8400, 8401, 3050, 4000, 5000, 5001 (5000/5001 mají iptables DROP), 3060.
chráněné iptables: 22 (2 IP), 3306 (188.245.255.117 + docker), 5000/5001 (docker only).

## Doporučené pořadí nápravy (vše prod infra → schvaluje Vladimir)
1. PGP-058 deploy hotfixu (relé routy) — kvůli H2 nejvyšší prio.
2. H1: restrikce/odpojení container SSH portů + PermitRootLogin no + PasswordAuthentication no.
3. H2: API porty (8400/8401/6001-3/4000) jen přes NPM, ne 0.0.0.0.
4. M1: 3306 bind local. M2: backup coverage + restore test.
5. NPM dead entries + secrets sken (samostatné tasky).
