# 25 -- Prod Runtime Drift Audit 2026-05-03

> Read-only audit of `server prod` (`pgepilot.cz`) against `lokal` and
> `GitHub upstream`.
> Last updated: 2026-05-03.

## Scope

This pass checked:

- `server prod`: host services, Docker containers, Nginx Proxy Manager routes,
  public HTTPS endpoints, selected runtime git states, PM2 status, and safe file
  hashes.
- `lokal`: nested repositories under `/Users/vladimiradam/projekty AI/pgepilot`
  after `git fetch --all --prune`.
- `GitHub upstream`: `origin/main` and each local branch's configured upstream.

No production files were changed, no containers were restarted, no deploy was
performed, and no credential values were copied into this report. Raw evidence is
kept locally under:

`/Users/vladimiradam/projekty AI/pgepilot/.server-audit/2026-05-03-pgepilot-prod/`

## Executive Findings

1. `pgepilot_service` on `server prod` is a dirty git checkout at
   `main@8a48706b`, which is `Holbytlo/pgepilot-service origin/main`, plus live
   server-local changes in `app/routes_pge_control.php` and
   `src/PgePilot/Api/TaskController.php`. The diff is large:
   `1453 insertions, 1 deletion`.
2. `pgepilot_worker1`, `pgepilot_worker2`, and `pgepilot_worker3` are clean at
   `main@8a48706b`. Their hashes for the two modified API files differ from
   `pgepilot_service`, so API and workers are not running from the same tree.
3. Public `pgepilot.cz`, `app.pgepilot.cz`, and `oldapp.pgepilot.cz` now route to
   `pgepilot_app_main:3060`, not to the older PGE App process inside
   `pgepilot_servicedesk`. This corrects older documentation.
4. `pgepilot_app_main:1f39444` matches `Holbytlo/pge-app origin/main@1f39444`.
   The local `pge-app` workspace is on a separate dirty WIP branch and is not the
   production source of truth.
5. `pgepilot_servicedesk` contains two dirty server-local git checkouts:
   `/home/app2/pge-app` and `/home/app2/servicedesk`. Both are behind their
   current GitHub `origin/main` and both include uncommitted runtime changes.
6. `pgepilot_service_demo` and `pgepilot_demo_simulator` are old dirty
   `pgepilot-service` checkouts at `main@1727f603`, behind `origin/main` by 4
   commits and with 40 tracked files changed. They should not be treated as a
   clean copy of either prod or GitHub.
7. `demo.pgepilot.cz` and `control.pgepilot.cz` route to `server dev`
   `37.27.32.17`, not to the local prod demo containers. On this check
   `demo.pgepilot.cz` returned `HTTP 200`, while `control.pgepilot.cz` returned
   `HTTP 502`.
8. `web.pgepilot.cz` and `taskmanager.pgepilot.cz` returned `HTTP 502`;
   `simsb.pgepilot.cz` failed at the HTTP check. These look like stale or broken
   proxy entries.

## Runtime Matrix

| Surface | `server prod` reality | `GitHub upstream` | `lokal` reality | Drift |
|---|---|---|---|---|
| `pgepilot.cz`, `app.pgepilot.cz`, `oldapp.pgepilot.cz` | NPM routes to `pgepilot_app_main:3060`; public bundle `/assets/index-GD72iBtP.js?v=20260430-ote-rules` | `Holbytlo/pge-app origin/main@1f39444`; matches image tag | `pge-app` is `codex/local-pge-app-wip-20260427@e0e7928`, dirty, ahead 32 / behind 3 vs `origin/main` | Prod matches GitHub main, not local WIP |
| `app2.pgepilot.cz` | NPM routes to `pgepilot_app2:3060`; public bundle `/assets/index-h8Jisf86.js` | Image provenance is not current `origin/main`; inspected image is legacy `sbtunnel3`/image-id runtime | No matching active local source selected in this pass | Legacy runtime; do not treat as canonical |
| `pgepilot_app_prod` | Container running but no public NPM route found in the checked proxy map | Image tag `782b8cc`; commit exists on demo/local branches, not as ancestor of `pge-app origin/main` | Local `pge-app` WIP contains branch history but is not clean | Running unused/standby old app image |
| `demo.pgepilot.cz` | NPM routes to `37.27.32.17:80`; public bundle `/assets/index-CzgxPtxN.js` | Not served from `pgepilot_app_main` or local prod `pgepilot_demo_app` | Demo branches exist in `pge-app`; not reconciled here | Public demo is `server dev`-backed |
| `pgepilot_demo_app` | Container running with tag `demo-prod-4a8c319`, bundle `/assets/index-lajybd5F.js` | Commit `4a8c319` is on demo branches, not `origin/main` | Local `pge-app` has demo branch refs | Running but not the public `demo.pgepilot.cz` target |
| `api.pgepilot.cz`, `service.pgepilot.cz` | `pgepilot_service:/var/www/html main@8a48706b`, dirty 2 tracked files, plus backup files | `Holbytlo/pgepilot-service origin/main@8a48706b` | `pgepilot-service` is `codex/local-pgepilot-service-wip-20260425@0e9152c`, dirty, ahead 6 / behind 3 vs `origin/main` | Live server-local API diff not in GitHub main or local branch as-is |
| `worker.pgepilot.cz`, `worker2.pgepilot.cz`, worker3 | Workers clean at `main@8a48706b`; public worker roots return `HTTP 403` | Match `Holbytlo/pgepilot-service origin/main@8a48706b` | Local service branch is separate WIP | Workers are clean, but differ from dirty API container |
| `pgepilot_service_demo`, `pgepilot_demo_simulator` | Dirty `main@1727f603`, 40 tracked files changed; demo/auth runtime files also present | `1727f603` is behind `pgepilot-service origin/main` by 4 commits | Local service branch is different WIP | Large server-local/demo drift |
| `jobmanager.pgepilot.cz`, `simsb.pgepilot.cz` | `pgepilot_jobmanager:/home/app main@4346047`, clean, PM2 `jobmanager` online; root URL returns `404`, `simsb` HTTP check failed | Matches `Holbytlo/pgepilot-js origin/main@4346047` | `pgepilot-js` WIP branch `bd6b5df`, ahead 1 vs `origin/main`, clean | Code clean on prod; route health needs separate check |
| `sd.pgepilot.cz`, `servicedesk.pgepilot.cz` | PM2 `servicedesk` online; `/home/app2/servicedesk main@5630af4`, dirty 11 tracked files and untracked auth/cloud-inventory files | `Holbytlo/servisdesk origin/main@d2ba76e`; server base behind by 3 | `servisdesk` is `codex/local-servisdesk-wip-20260427@01d30ff`, clean, ahead 1 / behind 2 vs `origin/main` | Dirty server-local servicedesk changes need reconciliation |
| `pgepilot_servicedesk:/home/app2/pge-app` | PM2 `pge-app` online; checkout `main@3d7e6bb`, dirty 5 tracked files, behind `pge-app origin/main` by 4 | Current `pge-app origin/main@1f39444`; public app no longer routes here | Local `pge-app` WIP branch is dirty | Dirty legacy/side runtime, not current public app |
| `auth.pgepilot.cz` | `pgepilot_auth_srv:1.0` runs `/app/auth_srv.js`, returns `HTTP 401` at root | File hashes match `Holbytlo/pgepilot-service origin/main` under `infra/auth_srv/*` | `pge-auth` is a separate WIP repo and not this legacy container's source | Auth runtime matches service repo, not local `pge-auth` branch |

## NPM / Public Endpoint Check

| URL | Backend from NPM | HTTP result | Note |
|---|---|---|---|
| `https://pgepilot.cz/` | `pgepilot_app_main:3060` | `200` | Public app shell |
| `https://app.pgepilot.cz/` | `pgepilot_app_main:3060` | `200` | Public app shell |
| `https://oldapp.pgepilot.cz/` | `pgepilot_app_main:3060` | `200` | Alias to current app image |
| `https://app2.pgepilot.cz/` | `pgepilot_app2:3060` | `200` | Legacy app2 runtime |
| `https://api.pgepilot.cz/api/v2/health` | `pgepilot_service:80` | `200` | API health OK |
| `https://service.pgepilot.cz/api/v2/health` | `pgepilot_service:80` | `200` | API alias health OK |
| `https://worker.pgepilot.cz/` | `pgepilot_worker1:80` | `403` | Backend reachable, root forbidden |
| `https://worker2.pgepilot.cz/` | `pgepilot_worker2:80` | `403` | Backend reachable, root forbidden |
| `https://sd.pgepilot.cz/` | `pgepilot_servicedesk:3050` | `200` | ServiceDesk reachable |
| `https://servicedesk.pgepilot.cz/` | `pgepilot_servicedesk:3050` | `200` | ServiceDesk alias reachable |
| `https://auth.pgepilot.cz/` | `pgepilot_auth_srv:4000` | `401` | Auth service reachable; root requires auth |
| `https://jobmanager.pgepilot.cz/` | `pgepilot_jobmanager:5000` | `404` | Service reachable but root route missing |
| `https://demo.pgepilot.cz/` | `37.27.32.17:80` | `200` | Served from `server dev` |
| `https://control.pgepilot.cz/` | `37.27.32.17:5000` | `502` | Broken at check time |
| `https://web.pgepilot.cz/` | `pgepilot.cz:8000` | `502` | Dead legacy proxy |
| `https://taskmanager.pgepilot.cz/` | `pgepilot.cz:5500` | `502` | Dead legacy proxy |
| `https://simsb.pgepilot.cz/` | documented as jobmanager sim route | `000ERR` | Not reachable in HTTP check |

## Recommended Next Steps

### Follow-up Preservation

The first recommended step was completed after this audit:

- repo: `Holbytlo/pgepilot-service`
- branch: `codex/reconcile-pgepilot-service-prod-20260503`
- commit: `3478d6f`
- scope: the two dirty tracked files from `pgepilot_service:/var/www/html`
  were copied read-only into a fresh worktree from `origin/main@8a48706b`
- production impact: none; no prod file write, deploy, or restart

Validation for that preservation branch:

- candidate file hashes match the live `server prod` files
- `git diff --check`: pass
- secret-pattern scan over the candidate diff: no findings
- `php -l` inside the live `pgepilot_service` container for both files: pass

Draft PR creation was attempted through the GitHub app, but the connector failed to
initialize and local `gh` was not authenticated. GitHub returned this manual PR URL
after push:

`https://github.com/Holbytlo/pgepilot-service/pull/new/codex/reconcile-pgepilot-service-prod-20260503`

The second preservation step was completed for `pgepilot_servicedesk` runtime
source trees:

`Holbytlo/pge-app`:

- branch: `codex/reconcile-pge-app-servicedesk-prod-20260503`
- commit: `8c147e3`
- base: server checkout base `3d7e6bb`
- scope: 5 dirty tracked files from
  `pgepilot_servicedesk:/home/app2/pge-app`; untracked `dist.*` backup
  artifacts were not copied
- validation: candidate hashes match live files, `git diff --check` passed,
  stricter secret-pattern scan produced no direct-secret findings, `npm ci`
  passed, `npm run build` passed
- manual PR URL:
  `https://github.com/Holbytlo/pge-app/pull/new/codex/reconcile-pge-app-servicedesk-prod-20260503`

`Holbytlo/servisdesk`:

- branch: `codex/reconcile-servisdesk-prod-20260503`
- commit: `6316c7d`
- base: server checkout base `5630af4`
- scope: 11 dirty tracked files plus 7 new auth/cloud-inventory code files from
  `pgepilot_servicedesk:/home/app2/servicedesk`; `.env.bak-*` and `*.bak-*`
  backup files were not copied
- validation: candidate hashes match live files, `git diff --check` passed,
  sensitive scan found only password variable reads in cloud-inventory code with
  no direct password literals, `npm ci` passed in `server` and `web`,
  `npm run build` passed in both `server` and `web`
- manual PR URL:
  `https://github.com/Holbytlo/servisdesk/pull/new/codex/reconcile-servisdesk-prod-20260503`

The GitHub app still failed to initialize when opening draft PRs, and local `gh`
remains unauthenticated. No production files were changed, no deploy was
performed, and no services or containers were restarted during either preservation
step.

The third preservation step was completed for demo/service-demo `pgepilot-service`
runtime trees:

`pgepilot_service_demo`:

- repo: `Holbytlo/pgepilot-service`
- branch: `codex/reconcile-pgepilot-service-demo-prod-20260503`
- commit: `a5f48c4`
- base: server checkout base `1727f603`
- scope: 40 dirty tracked files plus safe untracked runtime code files
  (`app/pge_auth_bridge.php`, `app/user_preferences.php`, `infra/auth_srv/*`,
  `scripts/bootstrap_runtime.php`, `scripts/demo_sb_lite.php`)
- excluded: `app/runtime_secrets.local.php`, `app/runtime_secrets.php`, and other
  credential/secret runtime files
- validation: candidate hashes match live files, `git diff --check` passed,
  secret-pattern classification found no direct quoted secrets, `php -l` passed
  for changed PHP files inside the live container, and `composer validate
  --no-check-publish` passed inside the live container
- manual PR URL:
  `https://github.com/Holbytlo/pgepilot-service/pull/new/codex/reconcile-pgepilot-service-demo-prod-20260503`

`pgepilot_demo_simulator`:

- repo: `Holbytlo/pgepilot-service`
- branch: `codex/reconcile-pgepilot-demo-simulator-prod-20260503`
- commit: `a9880ff`
- base: server checkout base `1727f603`
- scope: 40 dirty tracked files plus safe untracked runtime code files
  (`infra/auth_srv/*`, `scripts/bootstrap_runtime.php`, `scripts/demo_sb_lite.php`)
- excluded: `app/runtime_secrets.local.php`, `app/runtime_secrets.php`, and other
  credential/secret runtime files
- validation: candidate hashes match live files, `git diff --check` passed,
  secret-pattern classification found no direct quoted secrets, `php -l` passed
  for changed PHP files inside the live container, and `composer validate
  --no-check-publish` passed inside the live container
- manual PR URL:
  `https://github.com/Holbytlo/pgepilot-service/pull/new/codex/reconcile-pgepilot-demo-simulator-prod-20260503`

No production files were changed, no deploy was performed, and no services or
containers were restarted during this preservation step.

1. Preserve the `pgepilot_service` dirty prod diff into a clean local branch from
   `Holbytlo/pgepilot-service origin/main`, review it against current local WIP,
   then decide whether to PR it or revert/redeploy prod to GitHub main.
   Status: done as `3478d6f` on branch
   `codex/reconcile-pgepilot-service-prod-20260503`; PR still needs to be opened
   manually or once GitHub app/`gh` auth works.
2. Do not deploy a whole local `pgepilot-service` file over prod until the
   `pgepilot_service` server-local diff is classified. The API container already
   has live changes that the workers do not have.
3. Reconcile `pgepilot_servicedesk:/home/app2/servicedesk` and
   `pgepilot_servicedesk:/home/app2/pge-app` as separate tasks. Exclude env and
   backup files, classify auth/cloud-inventory changes, and only then decide what
   belongs in GitHub.
   Status: preservation branches are pushed as `8c147e3` (`pge-app`) and
   `6316c7d` (`servisdesk`); review/merge decisions remain open.
4. Decide whether `demo.pgepilot.cz` should intentionally stay on `server dev`
   `37.27.32.17` or move to a prod demo container. Current prod demo containers
   are not the public demo route.
   Status: prod demo container runtime diffs are preserved as `a5f48c4`
   (`pgepilot_service_demo`) and `a9880ff` (`pgepilot_demo_simulator`);
   routing/product decision remains open.
5. Clean or explicitly label stale NPM proxy entries (`web`, `taskmanager`,
   `simsb`, and older legacy hosts) after confirming they are not used.
6. Keep `pgepilot_app_main:1f39444` as the canonical current public app until a
   new app deploy is intentionally built from a chosen branch.
