# 19 -- PgePilot Auth Migration Plan

> Approved target architecture for PgePilot-owned authentication and migration away from ERP/PMO auth.
> Last updated: 2026-04-19

---

## Decision Summary

Approved direction:

- build a dedicated `PgePilot Auth` service owned by the PgePilot stack
- do **not** use ERP / PMO auth as the long-term identity source
- migrate `pge-app`, `sb-manager`, and `servisdesk`
- apply the same model in both dev and production
- use the safest browser model: server-side sessions with `HttpOnly` cookies
- do **not** send login tokens in URL query params
- do **not** store access or refresh tokens in `localStorage`
- keep SmartBox provisioning, SmartBox tokens, and local SB login completely separate

---

## Security Requirements

The target must satisfy these security properties:

- browser never sees reusable access or refresh tokens
- session cookies are `HttpOnly`, `Secure`, `SameSite=Lax` or stricter where compatible
- app cookies are host-only, not shared across all subdomains
- state-changing requests use CSRF protection
- login, logout, password change, and admin changes are audit-logged
- sessions are opaque random IDs stored server-side, not self-contained browser JWTs
- session fixation is prevented by rotating session IDs after login and privilege changes
- dev uses the same flow as production, with separate environment, secrets, and data

---

## Target Components

### 1. `pgepilot-auth`

New dedicated auth service for human users.

Responsibilities:

- login UI
- password auth
- optional future MFA / 2FA
- central user directory for PgePilot human users
- grants / permissions source of truth
- central SSO session on auth domain
- one-time authorization code issuance
- server-to-server code exchange for apps
- session / audit / password-reset lifecycle

Non-responsibilities:

- SmartBox machine auth
- SmartBox provisioning tokens
- local offline SB login
- device-to-device auth

### 2. App backends

Each app keeps its own local app session:

- `pgepilot-service` for `pge-app`
- `sb-manager` backend
- `servisdesk` backend

Each backend is responsible for:

- redirecting unauthenticated users to `pgepilot-auth`
- validating callback `code` + `state`
- exchanging code server-to-server
- creating local app session
- setting app-local secure cookie
- translating central identity into app-local authorization context

### 3. Browser

Browser stores only secure cookies:

- one auth-domain SSO cookie on the final approved `pgepilot-auth` public host
- one app session cookie per app host

Browser does **not** store:

- bearer tokens in URL
- refresh tokens in URL
- access tokens in `localStorage`
- refresh tokens in `localStorage`

---

## Recommended Domain Layout

### Production

- production public auth hostname is **TBD** and must be approved explicitly during design freeze
- do **not** assume `auth.pgepilot.cz` is free for reuse
- current runtime docs still treat `auth.pgepilot.cz` as legacy `pgepilot_auth_srv` / SmartBox auth infrastructure
- reusing `auth.pgepilot.cz` is allowed only after explicit SmartBox cutover / decommission work confirms that no remaining boxes depend on it
- existing app domains stay where they are today
- each app gets its own callback path on its own host

### Development

Minimum acceptable dev target:

- separate dev auth instance
- separate dev database
- separate secrets
- same redirect + cookie flow as production

Recommended dev hostnames:

- `auth.dev.pgepilot.cz` for shared dev
- local HTTPS aliases for local work, for example:
  - `auth.pgepilot.localhost`
  - `app.pgepilot.localhost`
  - `sb-manager.pgepilot.localhost`
  - `servisdesk.pgepilot.localhost`

Why:

- cookie behavior must match production as closely as possible
- auth that only works in prod but not in dev is unacceptable for this migration

---

## Session Model

### Central auth session

After successful login on the final approved `pgepilot-auth` public host:

- `pgepilot-auth` creates an opaque server-side session
- browser gets an auth-domain cookie such as `pge_auth_sso`
- cookie is `HttpOnly`, `Secure`, host-only, `Path=/`
- cookie value is only a random session ID

### App session

After authorization-code exchange:

- app backend creates its own opaque app session
- browser gets app-specific cookie such as:
  - `pge_app_session`
  - `sbm_session`
  - `sd_session`
- app cookie is `HttpOnly`, `Secure`, host-only, `Path=/`
- app cookie contains only opaque session ID

### Why two session layers

This gives:

- SSO across apps without sharing one broad cookie everywhere
- lower blast radius if one app is compromised
- simpler per-app logout and authorization
- safer isolation for audits

---

## Login Flow

### Target flow

1. User opens app.
2. App backend sees no local app session.
3. App backend redirects browser to `https://<pgepilot-auth-public-host>/login` with:
   - `client_id`
   - `redirect_uri`
   - `state`
   - optional `code_challenge` if PKCE is used
4. User logs in on `pgepilot-auth`.
5. `pgepilot-auth` creates central SSO session cookie.
6. `pgepilot-auth` redirects browser back to app callback with one-time `code` and `state`.
7. App backend validates `state`.
8. App backend exchanges `code` with `pgepilot-auth` over server-to-server call.
9. App backend receives user identity + grants + permission data from trusted backend channel.
10. App backend creates local app session and sets app cookie.
11. Browser continues only with cookies.

### Explicitly forbidden target behavior

The final design must not:

- redirect browser with `pge_token`
- redirect browser with `pge_refresh`
- ask frontend JS to store refresh token
- require frontend JS to attach bearer tokens to ordinary browser API calls

---

## User and Authorization Model

### Canonical human identity

`pgepilot-auth` should own canonical human users for PgePilot.

Initial migration sources:

- `cp_users`
- `cp_user_grants`
- current operator accounts from ERP-side `pge-auth` `users`
- current operator role assignments from ERP-side `roles`
- effective operator permission layer derived from `role_permissions` and `user_permission_overrides`

Why this must be explicit:

- `pge-app` currently authenticates against `cp_users` and authorizes through `cp_user_grants`
- current operator access for `sb-manager` and `servisdesk` depends on ERP-side `pge-auth` identities and RBAC
- a correct migration must inventory and map both populations before cutover

These sources should be imported, merged, or remapped into auth-owned tables so PgePilot auth stops depending on ERP identity.

### Authorization data

Central auth should own:

- user accounts
- password hashes
- roles
- grants
- app permissions
- session inventory
- auth audit log

### App-specific authorization adapters

Even after migration, apps still differ:

- `pge-app` uses grants like `SYSTEM`, `DOMAIN`, `CP`
- `sb-manager` maps central permissions to `readonly`, `ops`, `admin`
- `servisdesk` checks allowed roles and/or explicit permissions

Therefore:

- central auth is identity + permission source
- apps remain responsible for local authorization mapping

---

## API Contract For `pgepilot-auth`

Minimum endpoints:

- `GET /login`
- `POST /auth/login`
- `POST /auth/logout`
- `GET /auth/authorize`
- `POST /auth/token`
- `GET /auth/session`
- `POST /auth/change-password`
- `POST /auth/request-password-reset`
- `POST /auth/reset-password`
- `GET /health`

Notes:

- `GET /auth/authorize` issues one-time auth code for browser redirect flow
- `POST /auth/token` is used only for backend code exchange, not for browser token delivery
- if internal signed assertions are needed between services, keep them backend-only and short-lived

---

## Integration Per App

### `pge-app`

Target changes:

- replace current `POST /api/v2/auth/login` browser form flow
- app login screen can remain as branded entry page, but it should redirect to central auth
- remove browser-managed bearer token from ordinary SPA requests
- `pgepilot-service` should resolve user from app session cookie instead of frontend bearer token
- keep existing grant semantics unless and until business rules are redesigned

Pre-cutover prerequisite:

- inventory all accounts that still authenticate only through legacy plaintext `pgep_users` fallback
- either migrate them into canonical auth-owned accounts before cutover or force explicit password/account remediation during rollout
- do **not** assume current `pge-app` login is already clean just because `cp_users` + `cp_user_grants` exist

Compatibility bridge allowed only during migration:

- keep `/api/v2/auth/login` temporarily, but make it redirect or proxy into the new auth flow
- remove legacy direct JWT issuance from `pgepilot-service` after cutover

### `sb-manager`

Target changes:

- replace current ERP `pge-auth` dependency with `pgepilot-auth`
- remove URL token handoff and `localStorage` session handling
- backend should consume auth callback and create local session
- keep existing `readonly / ops / admin` app RBAC

### `servisdesk`

Target changes:

- same migration pattern as `sb-manager`
- local backend session instead of browser-stored tokens
- keep app-local allowed role / permission gate

---

## Current Blockers To Final Cutover

These items are not optional polish. They are the concrete reasons why production still cannot be switched safely.

1. Final public auth hostname is not approved yet.
2. `auth.pgepilot.cz` is still tied to legacy SmartBox auth, so it cannot be assumed free for human login.
3. Live `sb-manager` still points to `https://pmo.pgeuser.cz`, so production is still in ERP / PMO dependency mode.
4. Human identities are split across multiple sources:
   - `cp_users`
   - `cp_user_grants`
   - ERP-side `pge-auth` `users`
   - ERP-side RBAC tables
   - `pge-app` legacy `pgep_users` fallback
5. `pge-app` still contains legacy direct login behavior and is not yet clean for cutover.
6. There is no fully aligned dev environment yet that matches the final cookie-based login flow end to end.

Until all six blockers are closed, production cutover is not ready.

---

## Execution Roadmap

The roadmap below is the required path to final delivery, not just an architecture sketch.

### Phase 0 -- Freeze Scope And Final Auth Host

Goal:

- lock down what is being built and where it will live publicly

Required outputs:

- approved final public auth hostname
- explicit decision that production must stop depending on `pmo.pgeuser.cz`
- explicit deconfliction plan for legacy `auth.pgepilot.cz`
- approved redirect host allowlist for:
  - `pge-app`
  - `sb-manager`
  - `servisdesk`
- approved logout behavior:
  - app logout only
  - full SSO logout
  - session expiry behavior

Exit criteria:

- there is one approved public hostname for human auth
- there is one approved callback list
- there is one approved logout model

### Phase 1 -- Inventory And Identity Cleanup

Goal:

- establish the real source data that must be migrated into `pgepilot-auth`

Required outputs:

- inventory of all `cp_users`
- inventory of all `cp_user_grants`
- inventory of all current operator accounts in ERP-side `pge-auth`
- inventory of current ERP-side role and permission mappings
- inventory of all `pge-app` accounts still relying on `pgep_users`
- classification of each account:
  - migrate as-is
  - merge with another account
  - disable
  - force password reset / remediation

Mandatory decisions:

- canonical username / email format
- password migration strategy
- disabled / orphaned account policy
- role mapping per app

Exit criteria:

- no human account source is undocumented
- no legacy `pgep_users` account is left without explicit remediation plan
- there is a migration mapping for all app roles and permissions

### Phase 2 -- Build `pgepilot-auth` On Dev

Goal:

- deliver a real dev auth service with the final security model

Required outputs:

- new auth service skeleton
- auth database schema for:
  - users
  - credentials
  - roles / permissions / grants
  - sessions
  - auth codes
  - password reset
  - audit log
- login UI
- auth code issuance
- backend code exchange
- central SSO session
- local password change
- password reset flow
- audit logging
- rate limiting
- CSRF protection

Security acceptance:

- browser never receives reusable bearer tokens
- no auth token appears in URL
- no auth token is stored in `localStorage`
- cookies are `HttpOnly`, `Secure`, and host-scoped

Exit criteria:

- dev auth is functional end to end
- dev auth satisfies the target browser security model

### Phase 3 -- Integrate Apps On Dev

Goal:

- switch each app to the new auth flow while preserving app-local authorization behavior

Required order:

1. `pge-app`
2. `sb-manager`
3. `servisdesk`

Why this order:

- `pge-app` forces the cleanup of the hardest legacy login behavior first
- `sb-manager` and `servisdesk` then reuse the same callback/session pattern

Required app work:

For `pge-app`:

- replace browser login form flow with central redirect flow
- remove direct browser JWT login path
- remove legacy `pgep_users` fallback before cutover
- resolve user from secure app session

For `sb-manager`:

- replace ERP `pge-auth` dependency
- stop redirecting to `pmo.pgeuser.cz`
- keep app RBAC `readonly / ops / admin`

For `servisdesk`:

- replace ERP `pge-auth` dependency
- keep current role / permission gate semantics

Exit criteria:

- all three apps log in through dev `pgepilot-auth`
- all three apps use app-local secure cookies
- no app still needs PMO auth in dev

### Phase 4 -- Security Verification And Operational Readiness

Goal:

- prove the system is safe enough and operable enough for production

Required test set:

- login success
- login failure
- disabled user
- wrong role
- logout
- session timeout
- password reset
- CSRF rejection
- redirect allowlist rejection
- callback tampering rejection
- per-app rollback toggle

Operational readiness outputs:

- deployment procedure
- rollback procedure
- incident playbook
- metrics / logs checklist
- alerting for failed callback exchange, login spikes, session creation failures

Exit criteria:

- security checklist passes
- rollback works per app
- operations can deploy and rollback without manual improvisation

### Phase 5 -- Production Cutover

Goal:

- move production off ERP / PMO auth without user-facing outage

Required sequence:

1. deploy production `pgepilot-auth`
2. import / migrate approved user and permission set
3. verify auth host, callback URLs, and cookies on production domains
4. switch `pge-app`
5. switch `sb-manager`
6. switch `servisdesk`
7. watch logs, callback failures, session creation, and authorization denials

Hard gates before switch:

- final hostname conflict is resolved
- production secrets are separate from dev
- user import completed
- `pge-app` legacy fallback removed or explicitly disabled
- rollback flag available per app

Exit criteria:

- all three apps authenticate through `pgepilot-auth`
- no production app redirects to `pmo.pgeuser.cz`
- no production browser auth depends on URL or `localStorage` tokens

### Phase 6 -- Decommission Legacy Paths

Goal:

- remove the old auth dependencies so the migration is actually finished

Required cleanup:

- remove ERP / PMO auth dependency from all PgePilot apps
- remove PMO URLs from production env files
- remove query-param token handoff code
- remove `localStorage` auth token logic
- remove direct browser JWT issuance in `pgepilot-service`
- update documentation to state only `PgePilot Auth` as the live source of truth

Exit criteria:

- there is no remaining human-login dependency on ERP / PMO auth
- there is no live browser token handoff behavior left in PgePilot apps

---

## Definition Of Done

The migration is complete only when all statements below are true:

- `pge-app`, `sb-manager`, and `servisdesk` authenticate only through `PgePilot Auth`
- no production login redirect goes to `pmo.pgeuser.cz`
- no production human login depends on ERP-side identity data
- no browser access or refresh token is sent in URL
- no browser access or refresh token is stored in `localStorage`
- app sessions are host-scoped secure cookies
- SmartBox auth and provisioning remain fully separate

---

## Rollback Strategy

Each app should have a temporary feature flag:

- `AUTH_DRIVER=legacy`
- `AUTH_DRIVER=pgepilot-auth`

Rollback must be possible per app, not only globally.

This matters because:

- `pge-app`, `sb-manager`, and `servisdesk` have different authorization adapters
- a bug in one integration must not force a full auth outage for all apps

---

## Audit Checklist

Before go-live, verify:

- no login token is ever present in browser URL
- no access token is stored in `localStorage`
- no refresh token is stored in `localStorage`
- cookies are `HttpOnly`
- cookies are `Secure`
- cookies are scoped to exact host
- CSRF protection exists on state-changing routes
- session creation, logout, password change, and admin changes are logged
- idle timeout and absolute timeout are configured
- session revocation is possible from server side
- dev and prod are separated by secrets and data

---

## Non-Negotiable Boundaries

- SmartBox provisioning stays token-based and separate
- SmartBox RPC / machine auth stays separate
- local offline SB login stays separate
- ERP / PMO auth is not the long-term identity authority for PgePilot

---

## One-Line Summary

Build a dedicated `PgePilot Auth` service and migrate `pge-app`, `sb-manager`, and `servisdesk` to host-scoped `HttpOnly` cookie sessions via authorization-code style login, with no browser token handoff and no dependency on ERP / PMO auth.
