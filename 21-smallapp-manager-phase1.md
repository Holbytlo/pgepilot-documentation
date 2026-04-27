# 21 -- Smallapp Manager Phase 1

> Execution spec for the first production-ready slice of the `pgeusers` small-app platform.
> Last updated: 2026-04-19

---

## Goal

Build the first deployable version of a standalone `smallapp-manager` service that:

- acts as the admin hub on `smallapp.pgeuser.cz`
- keeps customer apps on `<slug>.smallapp.pgeuser.cz`
- uses a separate auth and user store from `pge-auth`, `sb-manager`, and `servisdesk`
- manages app registry, customer users, and access assignments
- generates a concrete proxy inventory for retiring public raw ports safely

This phase is about shipping the control plane first.

It is **not** about moving every existing mini-app immediately.

---

## Locked Decisions

The implementation must assume these points are already decided:

- admin hub lives on `smallapp.pgeuser.cz`
- customer app URL is `https://<slug>.smallapp.pgeuser.cz/`
- path aliases are optional compatibility only, not the primary model
- small-app auth is separate from operator auth
- app upstreams stay on their current internal ports in phase 1
- public raw ports are removed only after proxy validation per app
- apps that already operate as standalone products with their own domain/repo/runtime stay outside this stack

Additional implementation decision for phase 1:

- optional path aliases should default to **redirecting** to the canonical subdomain, not same-origin path proxying

Reason:

- redirect aliases preserve the subdomain isolation model
- they avoid forcing path-prefix compatibility on old SPAs
- they keep the long-term URL canonical from day one

---

## Phase 1 Scope

Deliver a standalone service named `smallapp-manager` with:

- local SQLite database
- server-side sessions with `HttpOnly` cookies
- admin login
- customer login
- app registry CRUD for the first migration wave
- customer user CRUD
- app access assignment per user
- admin audit log entries for auth and management actions
- generated reverse-proxy preview for wildcard subdomains and alias redirects
- minimal HTML admin dashboard
- minimal HTML customer dashboard

The first production migration wave still targets:

- 2 currently active small apps
- 2 new small apps

But the exact four apps remain operational rollout input, not a blocker for building the manager itself.

---

## Explicit Non-Goals

Phase 1 does **not** include:

- automatic Nginx reloads on the server
- wildcard DNS automation
- wildcard certificate automation
- migration of small apps into one shared frontend shell
- operator SSO reuse
- email-based password reset flow
- per-app embedded auth middleware inside legacy mini-app code
- immediate shutdown of all existing public ports
- migration of standalone products such as `LPS` into the small-app admin/auth model

---

## Data Model

### `apps`

Fields required now:

- `slug`
- `display_name`
- `upstream_host`
- `upstream_port`
- `status` (`draft`, `active`, `disabled`)
- `owner_name`
- `owner_email`
- `existing_public_url`
- `path_alias_enabled`
- `websocket_required`
- `sets_cookies`
- `supports_path_prefix`
- `notes`

### `users`

Fields required now:

- `email`
- `display_name`
- `role` (`admin`, `customer_user`)
- `password_hash`
- `is_disabled`

### `user_app_access`

Maps customer users to allowed apps.

### `sessions`

Stores opaque session IDs server-side.

### `password_reset_tokens`

Table exists now for forward compatibility, even if email reset is not in phase 1 UI.

### `auth_audit_log`

Must record:

- login success
- login failure
- logout
- app create
- user create
- access grant
- access revoke
- password reset
- user disable / enable

---

## HTTP Surface

### Human routes

- `GET /login`
- `POST /auth/login`
- `POST /auth/logout`
- `GET /admin`
- `GET /apps`
- `GET /admin/proxy-preview.txt`

### Admin actions

- `POST /admin/apps`
- `POST /admin/users`
- `POST /admin/access`
- `POST /admin/users/status`
- `POST /admin/users/password`
- `POST /account/password`

### Machine-friendly reads

- `GET /health`
- `GET /api/v1/state`
- `GET /api/v1/my/apps`

---

## Security Baseline

Phase 1 must already enforce:

- server-side session storage
- opaque random session IDs
- `HttpOnly` cookies
- `SameSite=Lax`
- `Secure` cookies in production
- host-only cookies
- session expiration
- session invalidation on logout
- password hashing with per-password salt
- CSRF token checks on authenticated form posts
- no browser bearer tokens
- no `localStorage` auth state

---

## Acceptance Criteria

Phase 1 is acceptable only if all of these are true:

1. A fresh instance can bootstrap an admin account and log in.
2. Admin can register at least one app with slug and upstream port.
3. Admin can create a customer user.
4. Admin can grant that user access to one app and not another.
5. Customer dashboard shows only assigned apps.
6. Canonical app URL shown to the customer is `<slug>.smallapp.pgeuser.cz`.
7. Optional path alias is represented as a redirect convenience, not as the primary URL.
8. Proxy preview output is concrete enough to wire subdomains on Nginx without re-inventing the routing plan.
9. Sessions survive ordinary page navigation but are removed on logout.
10. Admin actions are audit logged.

---

## Rollout Order After Merge

1. Deploy `smallapp-manager` behind internal port only.
2. Put `smallapp.pgeuser.cz` reverse proxy in front of it on `443`.
3. Seed the first 4 app records in the registry.
4. Validate generated proxy preview against real Nginx config conventions on `pgeusers`.
5. Proxy the first active existing app to `https://<slug>.smallapp.pgeuser.cz/`.
6. Run end-user validation of login, redirect, assets, websocket behavior, and cookies.
7. Only then close the old public raw port for that app.
8. Repeat per app.

---

## Recommended Next Engineering Steps

After phase 1 lands, the next slice should add:

- import/export for app inventory
- password reset workflow
- optional per-app launch notes / warnings
- deployment-state tracking (`inventory`, `proxied`, `raw-port-closed`)
- audit view in the admin UI
