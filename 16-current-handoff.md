# 16 -- Current Handoff

> Short operational handoff for the next chat/window.
> Last updated: 2026-04-24

This file is intentionally short. The previous long handoff from 2026-04-16 was archived to:

`../.archive/2026-04-24_docs_cleanup/pgepilot-documentation/16-current-handoff.pre-cleanup-2026-04-24.md`

Do not read that archived file by default. Use it only when historical debugging needs the detailed April 2026 SmartBox/history timeline.

---

## Read First

1. Workspace orientation: `../docs/ACTIVE_STATUS.md`
2. Documentation map: `INDEX.md`
3. Infrastructure: `05-infrastructure.md`
4. SmartBox/SB: `02-smartbox-sbc.md`, `14-smartbox-provisioning.md`, `17-smartbox-new-image-checklist.md`
5. Auth: `22-auth-schema-apps.md`, then `18-pgepilot-sluzby-prihlaseni.md`, then `19-pgepilot-auth-migration-plan.md`

## Current Workspace Reality

- `lokál` is a multi-repository workspace, not one git repo.
- Many nested repositories have local WIP changes. Do not reset or overwrite them.
- `server prod` is `pgepilot.cz`; read-only check on 2026-04-24 showed production Docker containers online.
- `server dev` is `ra-energity.cz`; read-only check on 2026-04-24 showed `sb-manager` and demo/simulator PM2 processes online.
- `GitHub upstream` is Holbytlo. Some repos need the Holbytlo SSH key; default SSH may fail.

## Current Open Topics

Use these as routing hints, not proof of final state:

| area | current source of truth |
|---|---|
| PgePilot cloud/API/history | `01-pgepilot-cloud.md`, `05-infrastructure.md`, `06-api-reference.md`, `07-entity-model.md`, `08-architecture-overview.md` |
| SmartBox runtime/provisioning | `02-smartbox-sbc.md`, `14-smartbox-provisioning.md`, `17-smartbox-new-image-checklist.md` |
| Auth and login | `22-auth-schema-apps.md`, `18-pgepilot-sluzby-prihlaseni.md`, `19-pgepilot-auth-migration-plan.md` |
| Roadmap / prioritization | `09-development-roadmap.md` |
| Security | `10-security.md` |

## Archive Boundary

Old handoffs, one-off plans, Superpowers docs, MP135 runbooks, and handoff ZIP manifests were moved to `.archive/2026-04-24_docs_cleanup/`.

AI agents should not inspect `.archive/` unless the user explicitly asks for a named historical artifact. The active docs above are the intended context path.

