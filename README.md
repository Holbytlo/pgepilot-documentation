# PGEPilot Documentation

Technical documentation for the PGEPilot ecosystem -- a photovoltaic (PV) monitoring and control platform by [Profi Green Energy](https://www.profi-green-energy.cz).

## Quick Start

- **AI/LLM agents**: Read [INDEX.md](INDEX.md) first -- it maps every file and tells you what to read next.
- **Humans**: Read [cs/README.md](cs/README.md) for Czech documentation.

## What This Covers

| Component | Description | Status |
|-----------|-------------|--------|
| **PgePilot Cloud** | Cloud monitoring, active plant fleet, 14 running Docker containers including proxy/demo/legacy app surfaces | Production |
| **SmartBox / SBC** | Edge Raspberry Pi, Modbus inverter control, reverse SSH | Production (monitoring), WIP (control) |
| **PGE App** | Vue 3 web frontend at app.pgepilot.cz | Production |
| **Mobile App** | React Native cross-platform client sharing API v2 with web | Implemented MVP, active development |
| **Infrastructure** | 3 servers (Hetzner), MariaDB, Docker, nginx | Production |
| **API v2** | 36+ REST endpoints, JWT auth, forecast, usage-based history resolver | Production |

## Documentation Structure

```
README.md                    <-- You are here
INDEX.md                     <-- AI entry point: file map
01-pgepilot-cloud.md         <-- Cloud platform overview
02-smartbox-sbc.md           <-- SmartBox edge system
03-pge-app-frontend.md       <-- Vue3 web application
04-mobile-app.md             <-- React Native mobile application (implemented MVP)
05-infrastructure.md         <-- Servers, Docker, databases, networking
06-api-reference.md          <-- All API endpoints
07-entity-model.md           <-- Domain model and database schema
08-architecture-overview.md  <-- System architecture and data flows
09-development-roadmap.md    <-- Current state and priorities
10-security.md               <-- Known risks and hardening plan
11-email-api.md              <-- Email API reference
16-current-handoff.md        <-- Short active handoff for next chat/window
17-smartbox-new-image-checklist.md <-- Practical checklist for new MP135/RPi image rollout
23-smartbox-fleet-registry.md <-- Canonical SmartBox label/alias/customer registry
cs/                          <-- Czech human-readable mirror
assets/                      <-- Wireframes, PDFs, diagrams
```

## Maintenance Rules

1. **English docs in root are the source of truth.** They are written AI-first (structured, explicit, self-contained).
2. **Czech docs in `cs/` are derived.** After any change to root docs, update the corresponding Czech file.
3. **Never commit credentials.** Use `[REDACTED]` placeholders and reference the sensitive local access notes under `/Users/vladimiradam/projekty AI/pristupy a servery/`.
4. **Keep INDEX.md up to date** whenever a file is added, removed, or significantly changed.
5. **Date every update** in the header of each document (format: YYYY-MM-DD).
6. **Keep historical handoffs out of the active read path.** Archive old
   session state under `../.archive/` and leave only short pointers in active
   docs.

## Credentials

All passwords, API keys, SSH keys, and tokens are **intentionally excluded** from this repository. They are stored in:

```
/Users/vladimiradam/projekty AI/pristupy a servery/PGEERP_Knowledge_Base.md
/Users/vladimiradam/projekty AI/pristupy a servery/pgepilot_server.md
```

Where you see `[REDACTED]` in a document, look up the actual value in those private files.

## Related Repositories

| Repo | Purpose |
|------|---------|
| [Holbytlo/pgepilot-service](https://github.com/Holbytlo/pgepilot-service) | PHP backend + API v2 + dataset/source policy resolver |
| [Holbytlo/pgepilot-js](https://github.com/Holbytlo/pgepilot-js) | Node.js job orchestrator |
| [Holbytlo/pgepilot-srv](https://github.com/Holbytlo/pgepilot-srv) | Auth server (JWT) |
| [Holbytlo/pge-app](https://github.com/Holbytlo/pge-app) | Vue3 frontend (PGE App) |
| [Holbytlo/sb](https://github.com/Holbytlo/sb) | SmartBox Python software |
| [Holbytlo/sb-manager](https://github.com/Holbytlo/sb-manager) | SmartBox device management |
