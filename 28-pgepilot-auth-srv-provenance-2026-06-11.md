# pgepilot_auth_srv runtime provenance + rebuild/rollback — 2026-06-11 (PGP-020)

Read-only forenzika běžícího `pgepilot_auth_srv` (kontejner na `pgepilot.cz`, public `:4000`). Provedl
claude_valVA_v2. ŽÁDNÁ změna/restart. Secrets nečteny.

## Závěr: runtime PŘESNĚ odpovídá origin/main, žádný drift

Nasazený `/app` (bez `.git`) byl spárován přes **git-blob hashe** souborů proti git repům.
Všechny tři soubory se shodují s `Holbytlo/pgepilot-service` `origin/main:infra/auth_srv/`:

| soubor | deployed git-blob | origin/main | shoda |
|---|---|---|---|
| auth_srv.js | `eaf7c0a` | `eaf7c0a` | ✅ |
| auth.js | `2ed9348` | `2ed9348` | ✅ |
| package.json | `1fc9ff0` | `1fc9ff0` | ✅ |

Na rozdíl od hlavní `pgepilot_service` aplikace (PGP-058, prod main nese zranitelný Route_Pgep.php),
auth_srv runtime **odpovídá committed zdroji** — žádný runtime-only drift.

## Image / build metadata

- Image: `pgepilot-auth-srv:1.0`, imageID `sha256:7d577f52be2851c6a6d848956c5f83a6aeb0acc2de0b1dac71e248e0edbc2e1c`
- Built: `2026-04-19T09:04Z` (compose project `pgepilot`, service `pgepilot_auth_srv`, compose 2.24.2)
- Entrypoint `docker-entrypoint.sh`, `CMD ["node","auth_srv.js"]`, WORKDIR `/app`, EXPOSE 4000
- Zdrojový commit: `d92f0d9` „Harden legacy auth service" (2026-04-19) = poslední commit měnící `infra/auth_srv/` na `origin/main`. Datum buildu == datum commitu → image je z tohoto commitu.

## Build context (origin/main:infra/auth_srv/Dockerfile)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY auth_srv.js auth.js ./
EXPOSE 4000
CMD ["node", "auth_srv.js"]
```

Služba: Express, port 4000, routy `/login` `/refresh-token` `/issue-smartbox-token` (POST),
`/verify-token` `/health` `/data` (GET). Závislosti: express, cors (auth.js = lokální modul).

## Rebuild postup (reprodukuje současný prod runtime)

```sh
# 1) čistý checkout zdroje
git -C <pgepilot-service> fetch origin
git -C <pgepilot-service> worktree add /tmp/authsrv-build origin/main   # nebo přímo commit d92f0d9
cd /tmp/authsrv-build/infra/auth_srv

# 2) build image (stejný tag)
docker build -t pgepilot-auth-srv:1.0 .

# 3) přes compose (projekt pgepilot) bez dotyku ostatních služeb
#    z adresáře s docker-compose.yml:
docker compose up -d --no-deps pgepilot_auth_srv
```

## Rollback

- Aktuální image je `pgepilot-auth-srv:1.0` (`sha256:7d577f52…`). Před jakýmkoli rebuild si ho ulož:
  `docker tag pgepilot-auth-srv:1.0 pgepilot-auth-srv:1.0-rollback-20260611` (a/nebo `docker save`).
- Návrat: `docker tag pgepilot-auth-srv:1.0-rollback-20260611 pgepilot-auth-srv:1.0 && docker compose up -d --no-deps pgepilot_auth_srv`.
- Zdrojový rollback bod: commit `d92f0d9` na `origin/main`.

## Pozn. — divergence auth.js na server-dev/devva

`origin/server-dev` i `origin/devva` nesou JINÝ `auth.js` (blob `3f0b328`, ne `2ed9348`). Prod běží
z `main`, ne ze server-dev. Pokud se má prod auth aktualizovat na server-dev verzi, je to vědomé
integrační rozhodnutí (diff auth.js main↔server-dev) — ne součást tohoto provenance ověření.

## Souvislost s bezpečností

`pgepilot_auth_srv` poslouchá na veřejném `0.0.0.0:4000` (viz PGP-032 audit). Provenance je čistá, ale
expozice portu zůstává bodem k řešení v PGP-032.
