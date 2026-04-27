# 22 -- Applications And Auth Logic Diagram

> Live graphical schema for human-user login across PgePilot services.
> Verified against code and live runtime on 2026-04-22.

---

## Scope

This document describes **human-user auth** for:

- `pge-app`
- `sb-manager`
- `servisdesk`

It does **not** describe SmartBox machine auth, provisioning tokens, or local SB login.

---

## 1. Current Live Topology

```mermaid
flowchart LR
  U["User / Browser"]

  subgraph APPS["Application hosts"]
    APP["app.pgepilot.cz\npge-app"]
    SBM["sb-manazer.ra-energity.cz\nsb-manager"]
    SD["sd.pgepilot.cz\nservisdesk"]
  end

  subgraph AUTH["Central human auth on api.pgepilot.cz"]
    LOGIN["/login\n/auth/code/exchange\n/auth/logout"]
    BRIDGE["/api/v2/auth/pge-login\nonly for pge-app"]
    USERS["pge_control.cp_users"]
    GRANTS["pge_control.cp_user_grants"]
    LEGACY["pgep_users\nlegacy password fallback"]
  end

  subgraph LOCAL["App-local sessions"]
    APPSESS["pge_app_session\nHttpOnly cookie"]
    SBMSESS["sb_manager_auth_session\nHttpOnly cookie"]
    SDSESS["servicedesk_auth_session\nHttpOnly cookie"]
  end

  U --> APP
  U --> SBM
  U --> SD

  APP -- "redirect to /login?app_id=pge-app&response_mode=code" --> LOGIN
  SBM -- "redirect to /login?app_id=sb-manager&response_mode=code" --> LOGIN
  SD -- "redirect to /login?app_id=servicedesk&response_mode=code" --> LOGIN

  LOGIN --> USERS
  LOGIN --> GRANTS
  LOGIN -. "fallback when cp_users password fails" .-> LEGACY

  LOGIN -- "one-time code redirect" --> APP
  LOGIN -- "one-time code redirect" --> SBM
  LOGIN -- "one-time code redirect" --> SD

  APP -- "POST /auth/code/exchange" --> LOGIN
  SBM -- "POST /auth/code/exchange" --> LOGIN
  SD -- "POST /auth/code/exchange" --> LOGIN

  APP -- "POST /api/v2/auth/pge-login" --> BRIDGE
  BRIDGE --> USERS
  BRIDGE --> GRANTS

  APP --> APPSESS
  SBM --> SBMSESS
  SD --> SDSESS
```

### Interpretation

- all three human-facing apps now start login on `api.pgepilot.cz`
- all three use `response_mode=code`
- browser receives **local app cookies**, not reusable access tokens in URL
- `pge-app` is the only app with an extra bridge step, because it still needs its own app JWT and grant payload

---

## 2. Generic Login Flow

```mermaid
sequenceDiagram
  participant User as "User / Browser"
  participant App as "App host"
  participant Auth as "api.pgepilot.cz"
  participant DB as "cp_users + cp_user_grants"

  User->>App: Open app
  App-->>User: 302 redirect to /login?app_id=...&response_mode=code
  User->>Auth: Open login form
  User->>Auth: Submit login/email + password
  Auth->>DB: Verify active user and load role
  Auth-->>User: Set central auth cookie
  Auth-->>App: Redirect back with one-time code
  App->>Auth: POST /auth/code/exchange
  Auth->>DB: Validate code / load active user
  Auth-->>App: access_token + refresh_token + user
  App-->>User: Set local HttpOnly session cookie
```

### `pge-app` extra step

```mermaid
sequenceDiagram
  participant App as "app.pgepilot.cz"
  participant Auth as "api.pgepilot.cz /auth/code/exchange"
  participant Bridge as "api.pgepilot.cz /api/v2/auth/pge-login"
  participant DB as "cp_users + cp_user_grants"

  App->>Auth: Exchange code
  Auth-->>App: central access_token
  App->>Bridge: POST /api/v2/auth/pge-login
  Bridge->>DB: Find matching cp_user
  Bridge->>DB: Load grants
  Bridge-->>App: app JWT + grants
  App-->>App: Create local pge_app_session
```

Meaning:

- `sb-manager` and `servisdesk` directly consume the central auth identity
- `pge-app` still converts that central identity into its own app token and grant model

---

## 3. Authorization Logic

```mermaid
flowchart TD
  START["Authenticated human user"]
  ROLE["Role from cp_users\nadmin / operator / viewer / obchodnik"]
  GRANT["Grants from cp_user_grants\nSYSTEM / DOMAIN / CP"]

  START --> ROLE
  START --> GRANT

  ROLE --> PGEAPP["pge-app"]
  GRANT --> PGEAPP
  ROLE --> SBM["sb-manager"]
  ROLE --> SD["servisdesk"]

  PGEAPP --> P1["Service mode:\nSYSTEM grant OR fallback admin/operator when no grants exist"]
  PGEAPP --> P2["Customer mode:\nDOMAIN or CP grant"]
  PGEAPP --> P3["Customer management:\nadmin or operator only"]

  SBM --> S1["admin -> admin + ops + readonly"]
  SBM --> S2["operator -> ops + readonly"]
  SBM --> S3["viewer -> readonly"]

  SD --> D1["Allowed by default:\nadmin, operator"]
  SD --> D2["Optional override:\nexplicit allowed permissions"]
```

### Practical rules

- `pge-app`
  - service/admin part is controlled by `role` and `SYSTEM` grant
  - customer part is controlled by `DOMAIN` / `CP` grants
- `sb-manager`
  - maps central role/permissions into local groups:
    - `pge_sbmanager_admin`
    - `pge_sbmanager_ops`
    - `pge_sbmanager_readonly`
- `servisdesk`
  - allows `admin` and `operator` by default
  - can also be extended by explicit permissions

---

## 4. Example: `holbytlo`

Current verified live identity:

- login: `holbytlo`
- role: `admin`
- grant: `SYSTEM`

Result:

- `pge-app` service/admin section: **yes**
- `sb-manager`: **yes**, full admin path
- `servisdesk`: **yes**
- `pge-app` customer-only section: **not by grant alone**, unless extra `DOMAIN` or `CP` grants are added

---

## 5. Important Caveat

Current central auth still contains a **legacy password fallback**:

- first try `pge_control.cp_users`
- if password check fails, try legacy `pgep_users`
- if legacy login matches and there is a mapped `cp_users` record, continue as mapped PgePilot user

That means the current live state is safer than the old token-in-URL flow, but not yet the clean final target from `19-pgepilot-auth-migration-plan.md`.
