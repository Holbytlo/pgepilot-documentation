# 24 -- Workspace / Git / KB Reconciliation 2026-05-03

> Read-only server checks and Git/KB reconciliation notes from PGP-052.
> Last updated: 2026-05-03.

## Scope

This pass reconciled evidence across:

- `lokál`: `/Users/vladimiradam/projekty AI/pgepilot`
- `server prod`: `pgepilot.cz`
- `server dev`: `37.27.32.17` / `pgepilot-dev.cz`
- SmartBox VPS/server-dev surface: `ra-energity.cz`
- `GitHub upstream`: Holbytlo repositories
- Local operational KB: `/Users/vladimiradam/projekty AI/pge-knowledgebase`

No production deploy, runtime file replacement, service restart, or server-side cleanup
was performed.

## GitHub / Local Repository State

All main nested project repositories fetched successfully on 2026-05-03, except
`pge-knowledgebase` with default SSH. KB fetch works with the Holbytlo Git key.

Primary local branches had no ahead/behind drift against their configured upstream
branch after fetch. Important local WIP remains:

| path | branch/head | status |
|---|---|---|
| `pge-app` | `codex/local-pge-app-wip-20260427@e0e7928` | dirty PGE App app-mode/forecast WIP |
| `pge-auth` | `codex/pgepilot-auth-code-flow-bootstrap-20260419@f7b4d36` | dirty `src/server.js` |
| `pgepilot-service` | `codex/local-pgepilot-service-wip-20260425@0e9152c` | dirty Forecast v2 API/migration WIP |
| `sb` | `codex/local-sb-wip-20260427@79bcf99` | dirty SmartBox/GoodWe/TOU WIP |
| `sb-manager` | `codex/local-sb-manager-wip-20260425@5662775` | dirty auth/admin/UI/ops-agent WIP |
| `.server-sync/pge-app-demo` | `pgepilot-dev-cleanup-20260502@db483da` | large dirty demo cleanup diff; not safe to auto-merge |
| `.worktrees/pge-app-app-modes` | `feature/app-modes@9618c15` | dirty app-mode WIP; branch behind `origin/main` by 4 commits |

Untracked `.DS_Store` noise was removed where it was clearly untracked.

## Server Prod Read-Only Check

Checked on 2026-05-03:

- Docker containers are online for PgePilot API, workers, apps, auth, demo, and
  nginx proxy.
- `https://api.pgepilot.cz/api/v2/health`, `https://app.pgepilot.cz/`,
  `https://demo.pgepilot.cz/`, and `https://pgepilot.cz/` returned `HTTP 200`.
- The inspected runtime paths inside `pgepilot_service`, `pgepilot_service_demo`,
  workers, `pgepilot_jobmanager`, and `pgepilot_servicedesk` did not expose `.git`
  directories. Treat server prod as runtime artifacts, not editable git checkouts.

Implication: do not use old docs that describe prod service/workers as dirty git
checkouts without a fresh check. Any prod change still requires the project deploy
safety rule: compare `GitHub upstream` or known base, current `lokál` candidate, and
current `server prod` runtime copy; create/identify a backup; inspect backup-to-candidate
diff before replacing anything.

## Server Dev / VPS Read-Only Check

Checked on 2026-05-03:

`37.27.32.17` / `pgepilot-dev.cz`:

- PM2 processes online: `pgepilot-app-modes-preview`, `pgepilot-public-preview`,
  and live-now daemons.
- `/home/limited/pgepilot-demo`, `/home/limited/pge-app-demo`,
  `/home/limited/pgepilot/sb-manager`, and `/home/limited/pgepilot-dev-npm`
  exist as runtime/source copies without `.git`.
- `https://pgepilot-dev.cz/` and `https://sb.pgepilot-dev.cz/` returned `HTTP 200`.

`ra-energity.cz`:

- PM2 processes online: `sb-manager`, `pgepilot-smartbox-sim`, and `sb999-demo-*`.
- `/opt/sb-router` is clean at `main@279b678`.
- `/opt/sbcsim` is clean at `main@87cc49d`.
- `/home/limited/sbcsim` is a separate clean checkout at `main@7780a8b`.
- `/opt/sb-manager` is dirty at `main@f1e8fd0`.
- `/home/limited/pgepilot-sb1-demo/sb` and
  `/home/limited/pgepilot-sb999-demo/sb` are dirty at `devva@296da87` and still
  contain AppleDouble `._*` git noise plus broken pack-index warnings.
- `https://sb-manazer.ra-energity.cz/` returned `HTTP 200`; `https://sb1.ra-energity.cz/`
  returned `HTTP 302`.

## Server-Local Changes Preserved In GitHub

The bounded `/opt/sb-manager` server-dev runtime changes were copied read-only from
server to a fresh local worktree from `origin/main` and preserved in GitHub:

- repo: `Holbytlo/sb-manager`
- branch: `codex/reconcile-sb-manager-server-dev-20260503`
- commit: `f084846`
- changed files:
  - `packages/sb-ops-agent/server.mjs`
  - `public/app.js`
  - `public/index.html`
  - `public/style.css`
  - `src/routes/admin.js`
- skipped server-local backup directories: `.backups/`, `backups/`

Validation:

- `git diff --check`: pass
- `node --check public/app.js`: pass
- `node --check src/routes/admin.js`: pass
- `node --check packages/sb-ops-agent/server.mjs`: pass
- `npm ci`: pass
- `npm test -- --test-reporter=spec`: 6/6 pass

Draft PR creation was not completed from Codex because the GitHub MCP app failed to
initialize and local `gh` is not authenticated. The branch is pushed and ready for a
manual or later automated draft PR.

## Server-Local Changes Not Auto-Preserved

The SB demo runtime checkouts on `ra-energity.cz` were not copied into GitHub in this
pass because they mix code, device config, many AppleDouble `._*` files, and broken
git object warnings. They need a separate reconciliation task that starts by copying
patches/configs into a clean worktree, classifying config vs code, and redacting any
site-local or credential-bearing content before commit.

The dirty `.server-sync/pge-app-demo` worktree was also not committed automatically
because it is a very large cleanup/deletion diff and is not proven to be the current
runtime source of truth.

## KB Reconciliation

The local PGE KB had a new `projects/pgepilot/sites/zbrasin-sb15.md` file. It was
sanitized to remove a plaintext access value and linked from:

- `AI_INDEX.md`
- `projects/pgepilot/README.md`
- `projects/pgepilot/sites/README.md`

The KB change should be committed and pushed with the Holbytlo Git key.

