# 16 -- Current Handoff

> Short operational handoff for the next chat/window.
> Last updated: 2026-05-03

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
6. Operational/project KB: `/Users/vladimiradam/projekty AI/pge-knowledgebase/AI_INDEX.md`

## Current Critical Handoff

Latest live SmartBox/GoodWe finding is in the dedicated operational KB:

- `/Users/vladimiradam/projekty AI/pge-knowledgebase/projects/pgepilot/handovers/2026-05-03-planicka-goodwe-parallel-charging.md`
- `/Users/vladimiradam/projekty AI/pge-knowledgebase/projects/pgepilot/sites/planicka-sb6.md`
- `/Users/vladimiradam/projekty AI/pge-knowledgebase/projects/pgepilot/inverters/goodwe/parallel-goodwe-et.md`

Summary:

- Planicka/SB6 is 2x GoodWe ET parallel.
- Both dongle IPs expose the same two Modbus unit IDs: `247` master/system and `1` slave.
- Slave `unit 1` returns `EXC:2` for normal ARM control registers (`47000/47120`, ECO/TOU slots, force-charge, peak-shaving, smart/parallel blocks).
- Live tests on 2026-05-03 did not find a safe non-EMS Modbus path for charging from grid.
- `47511/47512` were not written and remain blocked unless an explicit service test plan approves them.
- For `goodwe_parallel_et`, automatic OTE/spot control must not use non-zero `47120` or automatic grid charging. Use export-limit/stop-export only until a dedicated parallel strategy is verified.

## Current Workspace Reality

- `lokál` is a multi-repository workspace, not one git repo.
- Many nested repositories have local WIP changes. Do not reset or overwrite them.
- `server prod` is `pgepilot.cz`; focused read-only check on 2026-05-03 showed runtime artifacts without exposed `.git` in inspected containers and public health/app/demo URLs at `HTTP 200`.
- `server dev` has both `pgepilot-dev.cz` / `37.27.32.17` and the SmartBox/VPS surface on `ra-energity.cz`; verify the specific target for the task.
- `ra-energity.cz:/opt/sb-manager` dirty runtime changes were preserved to GitHub branch `codex/reconcile-sb-manager-server-dev-20260503` (commit `f084846`), but not deployed or merged.
- SB demo runtime checkouts under `/home/limited/pgepilot-sb*-demo/sb` remain dirty and need separate config/code/AppleDouble reconciliation.
- `GitHub upstream` is Holbytlo. Some repos need the Holbytlo SSH key; default SSH may fail.

## Current TaskAI Routing Hints

- PGP-051 is the live SB1/SB6 runtime merge and GoodWe protection strategy task.
- PGP-046 through PGP-050 cover PGE App safety/app-mode work and are pending Vladimir verification.
- PGP-038 through PGP-044 cover Forecast v2 model/calibration/API/UI work and remain in progress.
- PGP-001 and PGP-003 are pending verification; PGP-002 remains blocked until PGP-001 and the GoodWe parallel strategy are settled.

## Current Open Topics

Use these as routing hints, not proof of final state:

| area | current source of truth |
|---|---|
| PgePilot cloud/API/history | `01-pgepilot-cloud.md`, `05-infrastructure.md`, `06-api-reference.md`, `07-entity-model.md`, `08-architecture-overview.md` |
| SmartBox runtime/provisioning | `02-smartbox-sbc.md`, `14-smartbox-provisioning.md`, `17-smartbox-new-image-checklist.md`, plus `/Users/vladimiradam/projekty AI/pge-knowledgebase/AI_INDEX.md` for live plant knowledge |
| Auth and login | `22-auth-schema-apps.md`, `18-pgepilot-sluzby-prihlaseni.md`, `19-pgepilot-auth-migration-plan.md` |
| Roadmap / prioritization | `09-development-roadmap.md` |
| Security | `10-security.md` |
| Workspace/server/Git/KB reconciliation | `24-workspace-git-kb-reconciliation-2026-05-03.md` |

## Archive Boundary

Old handoffs, one-off plans, Superpowers docs, MP135 runbooks, handoff ZIP manifests,
broken demo worktree pointers, local cleanup artifacts, and broad runtime snapshots
were moved to `.archive/` cold storage.

AI agents should not inspect `.archive/` unless the user explicitly asks for a named historical artifact. The active docs above are the intended context path.
