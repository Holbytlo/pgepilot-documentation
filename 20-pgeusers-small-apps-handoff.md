# 20 -- Pgeusers Small Apps Handoff

> Proposed target model for small customer-facing apps currently running on raw ports on `pgeusers`.
> Last updated: 2026-04-19

---

## Decision Summary

Goal:

- keep the small apps alive
- stop depending on public raw ports
- give admin one place to manage access
- let customers open apps on HTTPS URLs, not `:3099`, `:3111`, etc.

Recommended direction:

- create one new public entrypoint: `smallapp.pgeuser.cz`
- use it as the admin-only control point for the small-app stack
- move app access behind reverse proxy on `443`
- keep each app on its current internal port on the server
- close public raw ports only after each app works through the new entrypoint

Important recommendation:

- **do not make path routing the only target model**
- support path aliases if needed, but default to **per-app subdomains**

Reason:

- `/sklenik` style routing is possible, but it is fragile for SPAs, static assets, cookies, redirects, and websockets
- `sklenik.smallapp.pgeuser.cz` is cleaner and safer
- one app on the same origin should not be able to interfere with all the others

---

## What Exists Today

Known pattern on `pgeusers`:

- several small apps run directly on host or Docker ports
- some already have a real domain and can later lose the raw port
- some are still reachable only or mainly by raw port

Examples already observed:

- `3090` = `lps`
- `3099` = `sklenik`
- `3111` = jalovy vykon visualization
- other app ports exist in the `8091` / `13010` / `13056` / `13060` / `13070` / `13080` family

Important classification update:

- `lps` was initially listed with the raw-port apps because it also uses `3090` on `pgeusers`
- after reviewing the actual runtime, `LPS` should be treated as a **standalone product**, not as a target small-app migration
- reason: it already has its own domain, repo, auth, database, and Docker runtime
- it may still share reverse-proxy conventions with the small-app stack, but it should not be folded into the small-app admin/auth scope

Security rule:

- small apps are different from infra ports
- public DB / Redis / Adminer style ports should stay closed
- app ports should be migrated behind HTTPS one by one, not hard-killed blindly

---

## Recommended Architecture

### 1. Public ingress

Expose only:

- `80`
- `443`

Everything else stays internal.

`smallapp.pgeuser.cz` should be the shared front door.

### 2. Admin layer

`smallapp.pgeuser.cz` should host:

- admin login
- app catalog / dashboard
- app registry
- customer user management
- mapping user -> allowed apps
- optional audit log of logins and password changes

Admin rule:

- root admin area is for you only
- customers should not see admin UI

### 3. App exposure model

Recommended primary model:

- `https://sklenik.smallapp.pgeuser.cz/`
- `https://jalovy.smallapp.pgeuser.cz/`

Optional compatibility alias:

- `https://smallapp.pgeuser.cz/sklenik`
- `https://smallapp.pgeuser.cz/jalovy`

Recommendation:

- keep path routing only as optional compatibility or launcher convenience
- use subdomains as the real long-term routing model

### 4. Upstream model

Apps do not need to move internally at first.

They can stay on:

- `127.0.0.1:3099`
- `127.0.0.1:3111`
- `127.0.0.1:8091`
- etc.

Reverse proxy maps public HTTPS hostnames to those internal upstreams.

### 5. Standalone product boundary

Not every customer-facing app on `pgeusers` belongs in the small-app stack.

Apps should stay **outside** the small-app platform when they already have:

- their own branded public domain
- their own repository
- their own auth model
- their own database and persistent storage
- their own deployment lifecycle

Current example:

- `LPS` / `lps-calc.cz` should stay a standalone product beside the small-app ecosystem
- it can keep using `pgeusers` and reverse proxy on `443`, but it should not be forced under `smallapp.pgeuser.cz`

---

## Why Subdomains Are Better Than Only Paths

### Path model: `smallapp.pgeuser.cz/sklenik`

Pros:

- one hostname
- simple catalog UX
- single TLS site

Cons:

- many apps assume `/`, not `/sklenik`
- static asset paths often break
- SPA router base path often breaks
- cookies and browser origin are shared across all apps
- one XSS bug in one app has larger blast radius
- websocket and redirect handling is more annoying

### Subdomain model: `sklenik.smallapp.pgeuser.cz`

Pros:

- cleaner isolation
- easier reverse proxying
- easier cookies
- easier redirects
- easier future per-app auth policy
- safer browser origin isolation

Cons:

- wildcard DNS and wildcard TLS are needed
- slightly more proxy records to manage

Decision:

- **admin hub on `smallapp.pgeuser.cz`**
- **customer apps primarily on `<slug>.smallapp.pgeuser.cz`**
- optional path aliases only where useful

---

## Auth and User Model

Recommended separation:

- do **not** mix these customer mini-app users with operator auth for `sb-manager` / `servisdesk`
- use a separate small-app auth/user store

Reason:

- operator tools and customer mini-apps are different security domains
- the customer app stack needs per-app assignments and simple support workflows

Recommended objects:

- `apps`
- `users`
- `user_app_access`
- `password_reset_tokens`
- `auth_audit_log`

Minimal permissions:

- `admin`
- `customer_user`

Admin can:

- create app entries
- create customer users
- assign user access to one or more apps
- disable user
- reset password

Customer can:

- log in
- see only assigned apps
- open only assigned apps

---

## Practical Migration Plan

### Phase 1 -- Inventory

For each raw-port app, record:

- slug
- current port
- app type
- owner
- whether it already has a domain
- whether it can live under path prefix
- whether it needs websocket support
- whether it sets cookies

### Phase 2 -- Shared entrypoint

Set up:

- `smallapp.pgeuser.cz`
- wildcard DNS for `*.smallapp.pgeuser.cz`
- wildcard TLS
- one reverse proxy layer for app routing

### Phase 3 -- Admin portal

Build a minimal admin tool:

- login
- list apps
- add app
- add user
- assign app access

This can start very small.

### Phase 4 -- First wave migration

Good first wave:

- the 2 currently active mini-apps
- the 2 new ones you want to add now

Explicit exclusion:

- do **not** pull `LPS` into this migration wave
- `LPS` is now classified as a separate standalone product

For each app:

1. keep current internal port
2. add new proxy route / subdomain
3. test end-user flow
4. only then close the public raw port

### Phase 5 -- Raw port shutdown

Once app-by-app validation passes:

- remove direct public access to that app port
- keep only HTTPS proxy access

---

## Suggested Naming

Admin hub:

- `smallapp.pgeuser.cz`

Per-app hostnames:

- `sklenik.smallapp.pgeuser.cz`
- `jalovy.smallapp.pgeuser.cz`
- `lps.smallapp.pgeuser.cz`

Optional short slugs:

- keep them lowercase
- no spaces
- ASCII only

---

## What I Would Avoid

- one big same-origin path-only stack as the final target
- public production use on random raw ports
- app-local password stores duplicated across mini-apps
- reusing operator auth for customer mini-apps
- closing all mini-app ports before proxy replacements exist
- dragging already-independent products such as `LPS` into the small-app control plane just because they happen to live on the same server

---

## First Thread Scope

The next implementation thread should decide only these points:

1. exact routing model:
   - subdomains only
   - or subdomains + path aliases
2. where the admin portal lives:
   - root `smallapp.pgeuser.cz`
   - or `smallapp.pgeuser.cz/admin`
3. which 4 apps are in wave 1:
   - current 2 active
   - plus the 2 new ones
4. whether auth is:
   - simple local DB auth for small apps
   - or a reusable auth service dedicated to small apps

My recommendation for that thread:

- choose `smallapp.pgeuser.cz` as admin hub
- choose `<slug>.smallapp.pgeuser.cz` as the real app URL
- allow `smallapp.pgeuser.cz/<slug>` only as optional convenience alias
- build a separate small-app auth/user store

---

## One-Line Summary

Build one admin-controlled HTTPS gateway for `pgeusers` small apps, keep apps on internal ports, expose them primarily as `<slug>.smallapp.pgeuser.cz`, and retire public raw ports only after each app is safely migrated behind the proxy.
