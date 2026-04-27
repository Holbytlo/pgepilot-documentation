# 18 -- PGEPilot služby přihlášení

> Live handoff for the shared operator login layer across `pge-auth`, `sb-manager`, and `servisdesk`.
> Last updated: 2026-04-19

> Important: this file describes the current live reuse of ERP-side `pge-auth`.
> The approved target architecture for PgePilot-owned auth is documented separately in `19-pgepilot-auth-migration-plan.md`.

---

## Scope

This document is about the current shared browser authentication model for:

- `pge-auth` as the central identity service
- `sb-manager`
- `servisdesk`
- redirect-based login handoff
- access/refresh token lifecycle
- per-service authorization
- logout behavior and future hardening questions

This document is **not** about:

- SB1 local user login in `sb/web_interface`
- SmartBox device provisioning token auth on the box itself
- customer-facing access/grants inside `pge-app`
- generic `AUTH_MODE=oidc` unless a service is explicitly switched to it

Why this boundary:

- `sb-manager` and `servisdesk` already use the same `pge-auth` browser login pattern
- `pge-auth` is the identity system; services consume tokens and apply their own authorization rules
- adjacent apps may have different auth models and should be documented separately unless explicitly verified

---

## Current Runtime Model (2026-04-19)

This section reflects today's deployment note together with the current repo code. The code confirms the wiring and auth flow. If someone needs VPS-level confirmation again, re-check deployed env/nginx directly.

### Central identity source: `pge-auth`

Current central auth behavior:

- `pge-auth` serves the browser login page at `/login`
- it supports password login via `/auth/web-login`
- it supports JSON/token login via `/auth/login`
- it exposes `/auth/verify`, `/auth/refresh`, and `/auth/logout`
- it can also create sessions from Microsoft identity via `/auth/microsoft-sync`
- issued access tokens include `role` and `permissions`
- refresh tokens are persisted server-side as bcrypt hashes

### `sb-manager`

Current production path should be treated as:

- `AUTH_MODE=pge-auth`
- `/api/v1/auth/start` redirects to central `pge-auth`
- `/api/v1/auth/verify`, `/auth/refresh`, `/auth/logout`, and `/auth/login` proxy to `pge-auth`
- request authentication uses Bearer token verification through `pge-auth`
- old nginx Basic Auth + fixed admin-header behavior is historical context, not current runtime guidance

### `servisdesk`

Current `servisdesk` auth shape:

- mounts central auth routes under `/api/auth/*`
- redirects browser login through `pge-auth`
- protects `/api` with `requirePgeOperator`
- frontend uses the same token handoff pattern as `sb-manager`

### What is no longer the current story

Do **not** treat these as current live state in future chats:

- `sb-manager` fronted by browser `HTTP Basic Auth`
- `sb-manager` live in `AUTH_MODE=proxy`
- "anyone who passes nginx popup becomes admin"

---

## Shared Browser Login Flow

1. User opens `sb-manager` or `servisdesk`.
2. Frontend asks the local backend whether auth is required via `/api/v1/auth/config` or `/api/auth/config`.
3. If auth is required, frontend redirects browser to the local start route and includes the current page URL as `redirect`.
4. Service backend builds a central login URL in the form `pge-auth /login?app_id=<service>&redirect=<same-origin-service-url>`.
5. User submits credentials on the `pge-auth` login page.
6. `pge-auth /auth/web-login` creates an access token and refresh token.
7. `pge-auth` redirects browser back to the service with `pge_token` and `pge_refresh` in query params.
8. Frontend consumes those query params, stores them in `localStorage`, removes them from the visible URL with `history.replaceState`, and verifies the session through `/auth/verify`.
9. If the access token is no longer valid, frontend calls `/auth/refresh`, stores the new access token, and retries verify.
10. Logout posts the refresh token to `/auth/logout`, clears local state, and redirects back into the login flow.

---

## Session and Token Model

### Access token

Current access token properties:

- JWT issued by `pge-auth`
- default TTL is `1h`
- payload includes `uid`, `email`, `name`, `role`, `permissions`, and audience derived from `app_id`
- services send it as `Authorization: Bearer ...`
- services currently verify it through `pge-auth /auth/verify`, not via their own cookie session

### Refresh token

Current refresh token properties:

- separate JWT with default TTL `2h`
- stored server-side as bcrypt hash in `refresh_tokens`
- stored client-side in browser `localStorage`
- used by `/auth/refresh` to mint a new access token
- current refresh flow returns a new access token, not a rotated refresh token

### Browser storage

Current browser session persistence in both `sb-manager` and `servisdesk`:

- access token in `localStorage`
- refresh token in `localStorage`
- cached user payload in `localStorage`
- immediate URL cleanup after handoff by removing `pge_token` and `pge_refresh`

### Logout semantics

Current logout behavior:

- browser posts `refresh_token` to `/auth/logout`
- `pge-auth` revokes refresh token state for the user
- frontend clears local storage and redirects back to login
- already-issued access tokens remain valid until their own expiry window closes

---

## Authorization Model

### `pge-auth`

`pge-auth` should be treated as the source of truth for:

- identity
- base role assignment
- permission resolution
- admin-side RBAC APIs for roles, permissions, user overrides, and related settings

### `sb-manager`

`sb-manager` keeps its own app-local RBAC surface:

- `readonly`
- `ops`
- `admin`

Those app roles are derived from central roles/permissions via local group mapping:

- admin roles/perms map to `pge_sbmanager_admin` and inherit lower groups
- ops roles/perms map to `pge_sbmanager_ops` and inherit readonly
- readonly roles/perms map to `pge_sbmanager_readonly`
- local middleware then collapses groups to one of `readonly` / `ops` / `admin`

### `servisdesk`

`servisdesk` uses a different authorization adapter:

- it does not use `sb-manager` group mapping
- it allows access based on configured allowed roles
- current default allowed roles are `admin`, `backoffice`, `superadmin`
- it can also allow access via explicit configured permissions
- all protected API routes run behind `requirePgeOperator`

### Practical implication

Identity is centralized, but authorization is still service-specific.

That means:

- `pge-auth` role/permission changes can affect services differently
- auth debugging must distinguish "token is valid" from "service accepted this user"
- future RBAC changes should be reviewed per service, not only in `pge-auth`

---

## Current Security Trade-offs

### 1. Token handoff uses redirect query params

Current web login returns:

- `pge_token`
- `pge_refresh`

in the redirect URL.

Impact:

- tokens are briefly present in the browser address bar
- tokens may leak via logs, screenshots, browser sync, copied URLs, or badly timed `Referer` propagation
- frontend cleanup happens quickly, but only after the redirect lands in the target app

Current mitigations:

- `pge-auth` sanitizes redirect hosts against an allowlist/base URLs
- service backends sanitize local return URLs to same-origin targets
- both frontends remove token query params immediately after handoff

### 2. Tokens live in `localStorage`

Current persistence is simple and reload-friendly, but XSS-sensitive.

Impact:

- successful XSS in `sb-manager` or `servisdesk` can read browser-stored tokens
- there is no `HttpOnly` cookie boundary today

### 3. Logout revokes refresh, not already-issued access tokens

Current logout is refresh-token revocation plus local browser cleanup.

Impact:

- stolen access tokens remain usable until expiry
- access-token TTL is therefore an important containment boundary

### 4. Shared login, different authorization adapters

Current shared auth layer reduces duplicate identity code, but app-specific authorization still differs.

Impact:

- `servisdesk` and `sb-manager` must be reviewed separately when central roles or permissions change
- one service can reject a user even though another accepts the same token

---

## Operational Guardrails

### Keep these concerns separate

Do not mix:

- operator web login via `pge-auth`
- SB1 local app login
- SmartBox provisioning token auth

These are different systems with different failure modes.

### Do not break provisioning

`POST /api/v1/provision/register` in `sb-manager` is a bearer-token provisioning endpoint and must remain outside interactive browser login.

It is part of bootstrap/provisioning flow, not operator sign-in.

### Do not reintroduce `AUTH_MODE=dev` in production

`sb-manager` still supports `dev` for local work, but production startup blocks it unless explicitly overridden.

### Do not turn services into separate identity silos

`sb-manager` and `servisdesk` should not grow their own password lifecycle or user stores unless there is an explicit architecture decision to do so.

---

## Future Work / Open Questions

1. Should browser token handoff stay query-param based, or move to a more defensive one-time code exchange / secure cookie session model?
2. Is `localStorage` acceptable for operator-facing tools, or should this stack move to `HttpOnly` cookies and a BFF/session approach?
3. What should global logout mean across multiple services sharing the same `pge-auth` identity layer?
4. Should `servisdesk` and `sb-manager` converge on one common permission vocabulary instead of separate adapters?
5. Is 2FA/MFA a near-term requirement for operator-facing services? No 2FA flow is visible in the current verified code paths.
6. Does incident response need explicit token/session introspection or revocation beyond refresh-token invalidation?

---

## Key Files For Follow-up

### Central auth

- `/Users/vladimiradam/projekty AI/pgepilot/pge-auth/src/server.js`

### `sb-manager`

- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/src/server.js`
- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/src/middleware/auth.js`
- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/src/middleware/rbac.js`
- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/src/lib/pgeAuth.js`
- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/src/routes/auth.js`
- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/src/routes/provision.js`
- `/Users/vladimiradam/projekty AI/pgepilot/sb-manager/public/app.js`

### `servisdesk`

- `/Users/vladimiradam/projekty AI/pgepilot/servisdesk/server/src/index.ts`
- `/Users/vladimiradam/projekty AI/pgepilot/servisdesk/server/src/lib/pgeAuth.ts`
- `/Users/vladimiradam/projekty AI/pgepilot/servisdesk/web/src/auth.ts`

---

## One-Line Summary

As of 2026-04-19, operator login for `sb-manager` and `servisdesk` is centered on `pge-auth`; browser sessions use redirect-based token handoff (`pge_token` / `pge_refresh`), `localStorage` persistence, proxied verify/refresh/logout, and per-service authorization layered on top of central identity.
