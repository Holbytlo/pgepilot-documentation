# INDEX -- Documentation Map

> **For AI/LLM agents.** Read this file first. Each entry has a one-line description so you know exactly what to read next.
>
> Last updated: 2026-04-12

---

## File Map

| # | File | What it contains |
|---|------|-----------------|
| -- | [README.md](README.md) | Project intro, maintenance rules, credential policy, related repos |
| 01 | [01-pgepilot-cloud.md](01-pgepilot-cloud.md) | Cloud platform: 7 Docker containers, 23 plants, task system, forecast, data sync, git repos |
| 02 | [02-smartbox-sbc.md](02-smartbox-sbc.md) | SmartBox edge: VPS, Raspberry Pi, 4 microservices, Modbus polling, TOU engine, SSH tunnels |
| 03 | [03-pge-app-frontend.md](03-pge-app-frontend.md) | Vue3 SPA: routing, components, theming, build/deploy, file structure |
| 04 | [04-mobile-app.md](04-mobile-app.md) | Planned React Native app: screens, entity model, shared API v2 |
| 05 | [05-infrastructure.md](05-infrastructure.md) | 3 servers, Docker containers, ports, firewall, DB overview, deploy procedures, backups |
| 06 | [06-api-reference.md](06-api-reference.md) | API v2 (36+ endpoints), legacy v1, Email API, RPC sb.* methods, external connectors |
| 07 | [07-entity-model.md](07-entity-model.md) | Domain model: new (pge_control) + legacy (pgepilot), DDL, mapping, separation rules |
| 08 | [08-architecture-overview.md](08-architecture-overview.md) | System diagrams, phase plan (A-D), data flows, legacy coexistence |
| 09 | [09-development-roadmap.md](09-development-roadmap.md) | Production state, completed milestones, priority 1-4 items with timelines |
| 10 | [10-security.md](10-security.md) | Critical/medium risks, hardening plan, RBAC status |
| 11 | [11-email-api.md](11-email-api.md) | POST /sendEmail: profiles, SMTP config, code examples |
| 12 | [12-forecasting.md](12-forecasting.md) | PV forecast, load forecast, weather, adaptive correction, planned Open-Meteo physical model |
| 13 | [13-connector-cache.md](13-connector-cache.md) | Connector self-governance: cache strategy, budget/rate limits, enable flags, ConnectorBudgetTrait |
| 14 | [14-smartbox-provisioning.md](14-smartbox-provisioning.md) | Kompletni provisioning noveho SB: flash, bootstrap, konfigurace, overeni, checklist |
| -- | [AUDIT_PROCEDURE.md](AUDIT_PROCEDURE.md) | Step-by-step server audit commands for verifying docs against reality |

## How to Use

- **Working on PgePilot cloud?** Start with `01`, then `06` (API), `07` (entities).
- **Working on SmartBox?** Start with `02`, then `14` (provisioning), `06` (RPC section), `07` (entities).
- **Installing a new SmartBox?** Read `14` (provisioning guide) — step by step from zero to production.
- **Working on the web app?** Start with `03`, then `06` (API).
- **Need infrastructure details?** Read `05`.
- **Need the big picture?** Read `08` (architecture), then `09` (roadmap).
- **Security audit?** Read `10`.

## Credential Policy

All credentials are `[REDACTED]`. Actual values are in:
- `zadani/pristupy a servery/PGEERP_Knowledge_Base.md` (private OneDrive)
- `zadani/pristupy a servery/pgepilot_server.md` (private OneDrive)
