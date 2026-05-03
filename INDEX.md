# INDEX -- Documentation Map

> **For AI/LLM agents.** Read this file first. Each entry has a one-line description so you know exactly what to read next.
>
> Last updated: 2026-05-03

---

## File Map

| # | File | What it contains |
|---|------|-----------------|
| -- | [README.md](README.md) | Project intro, maintenance rules, credential policy, related repos |
| 01 | [01-pgepilot-cloud.md](01-pgepilot-cloud.md) | Cloud platform: 7 Docker containers, 23 plants, task system, forecast, data sync, git repos |
| 02 | [02-smartbox-sbc.md](02-smartbox-sbc.md) | SmartBox edge: VPS, Raspberry Pi, 4 microservices, Modbus polling, TOU engine, SSH tunnels |
| 03 | [03-pge-app-frontend.md](03-pge-app-frontend.md) | Vue3 SPA: routing, components, theming, build/deploy, file structure |
| 04 | [04-mobile-app.md](04-mobile-app.md) | React Native app: screens, shared API v2, charts and detail parity with web |
| 05 | [05-infrastructure.md](05-infrastructure.md) | 3 servers, Docker containers, ports, firewall, DB overview, deploy procedures, backups |
| 06 | [06-api-reference.md](06-api-reference.md) | API v2 (36+ endpoints), history `usage` resolver, legacy v1, Email API, RPC sb.* methods |
| 07 | [07-entity-model.md](07-entity-model.md) | Domain model: new (pge_control) + legacy (pgepilot), dataset ownership, lineage, separation rules |
| 08 | [08-architecture-overview.md](08-architecture-overview.md) | System diagrams, raw -> derived data flows, usage-based history resolution, legacy coexistence |
| 09 | [09-development-roadmap.md](09-development-roadmap.md) | Production state, completed milestones, history-model normalization work, priority 1-4 items |
| 10 | [10-security.md](10-security.md) | Critical/medium risks, hardening plan, RBAC status |
| 11 | [11-email-api.md](11-email-api.md) | POST /sendEmail: profiles, SMTP config, code examples |
| 12 | [12-forecasting.md](12-forecasting.md) | PV forecast (forecast.solar + Open-Meteo physical model), load forecast, weather, adaptive EMA correction. Design spec: [assets/open-meteo-fve-navrh.md](assets/open-meteo-fve-navrh.md) |
| 13 | [13-connector-cache.md](13-connector-cache.md) | Connector self-governance: cache strategy, budget/rate limits, enable flags, ConnectorBudgetTrait |
| 14 | [14-smartbox-provisioning.md](14-smartbox-provisioning.md) | Kompletni provisioning noveho SB: flash, bootstrap, konfigurace, overeni, checklist |
| 17 | [17-smartbox-new-image-checklist.md](17-smartbox-new-image-checklist.md) | Prakticky checklist pro novy MP135/RPi image: build, flash, sit, bootstrap, connector, validation |
| 16 | [16-current-handoff.md](16-current-handoff.md) | Short current handoff and active context routing; points to latest Planicka/SB6 GoodWe parallel charging KB |
| 18 | [18-pgepilot-sluzby-prihlaseni.md](18-pgepilot-sluzby-prihlaseni.md) | Shared operator login across pge-auth, sb-manager, and servicedesk: redirect flow, tokens, RBAC, and risks |
| 19 | [19-pgepilot-auth-migration-plan.md](19-pgepilot-auth-migration-plan.md) | Approved target architecture for PgePilot-owned auth, secure session model, and migration plan for pge-app, sb-manager, and servicedesk |
| 20 | [20-pgeusers-small-apps-handoff.md](20-pgeusers-small-apps-handoff.md) | Recommended target model for small apps on pgeusers: admin hub, per-app routing, auth separation, and raw-port retirement |
| 21 | [21-smallapp-manager-phase1.md](21-smallapp-manager-phase1.md) | Phase-1 execution spec for the standalone smallapp-manager service, including scope, routes, security baseline, and rollout acceptance criteria |
| 22 | [22-auth-schema-apps.md](22-auth-schema-apps.md) | Graphical schema of current live auth across pge-app, sb-manager, and servisdesk, including login flow and authorization rules |
| 23 | [23-smartbox-fleet-registry.md](23-smartbox-fleet-registry.md) | Canonical SmartBox fleet registry: labels, aliases, customer/site mapping, expected inverter class, and current provisioning blockers |
| -- | [AUDIT_PROCEDURE.md](AUDIT_PROCEDURE.md) | Step-by-step server audit commands for verifying docs against reality |

## How to Use

- **Working on PgePilot cloud?** Start with `01`, then `06` (API), `07` (entities).
- **Working on SmartBox?** Start with `23` (fleet registry), then `02`, `17` (new image checklist), `14` (provisioning), `06` (RPC section), `07` (entities).
- **Installing a new SmartBox?** Read `17` first (operational checklist), then `14` (full provisioning guide).
- **Working on the web app?** Start with `03`, then `06` (API).
- **Need infrastructure details?** Read `05`.
- **Need the big picture?** Read `08` (architecture), then `09` (roadmap).
- **Continuing from another chat/window?** Read `16` first, then jump into the listed files and commands.
- **Security audit?** Read `10`.
- **Need auth map / login picture?** Read `22`, then `18`, then `19`.

## Archive Boundary

Old handoffs, one-off plans, MP135 runbooks, and handoff packages are in
`../.archive/2026-04-24_docs_cleanup/`. They are cold storage. Do not read
them by default; use the active docs above unless a task explicitly asks for
historical recovery.

## Credential Policy

All credentials are `[REDACTED]`. Actual values are in:
- `zadani/pristupy a servery/PGEERP_Knowledge_Base.md` (private OneDrive)
- `zadani/pristupy a servery/pgepilot_server.md` (private OneDrive)
