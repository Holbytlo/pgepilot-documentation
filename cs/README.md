# PGEPilot Dokumentace (CZ)

> Toto je ceska verze dokumentace. Hlavni (AI-first) verze je v korenovem adresari v anglictine.
> **Po kazde zmene v korenovem adresari aktualizujte i tento CZ mirror.**

Technicka dokumentace pro ekosystem PGEPilot -- platforma pro monitoring a rizeni fotovoltaickych instalaci od [Profi Green Energy](https://www.profi-green-energy.cz).

---

## Prehled komponent

| Komponenta | Popis | Stav |
|------------|-------|------|
| **PgePilot Cloud** | Cloudovy monitoring, 23 aktivnich plantu, 7 Docker kontejneru | Produkce |
| **SmartBox / SBC** | Edge Raspberry Pi, Modbus rizeni invertoru, reverzni SSH | Produkce (monitoring), Rozpracovano (rizeni) |
| **PGE App** | Vue 3 webovy frontend na app.pgepilot.cz | Produkce |
| **Mobilni app** | React Native cross-platform klient sdilejici API v2 s webem | Implementovane MVP, aktivni vyvoj |
| **Infrastruktura** | 3 servery (Hetzner), MariaDB, Docker, nginx | Produkce |
| **API v2** | 36+ REST endpointu, JWT autentizace, predikce, usage-based resolver historie | Produkce |

---

## Struktura dokumentace

```
cs/README.md                     <-- Jste tady
cs/01-pgepilot-cloud.md          <-- Cloudova platforma
cs/02-smartbox-sbc.md            <-- SmartBox edge system
cs/03-pge-app-frontend.md        <-- Vue3 webova aplikace
cs/04-mobile-app.md              <-- React Native mobilni aplikace (implementovane MVP)
cs/05-infrastructure.md          <-- Servery, Docker, databaze, sit
cs/06-api-reference.md           <-- Vsechny API endpointy
cs/07-entity-model.md            <-- Domenovy model a DB schema
cs/08-architecture-overview.md   <-- Architektura systemu a datove toky
cs/09-development-roadmap.md     <-- Aktualni stav a priority
cs/10-security.md                <-- Znama rizika a plan zabezpeceni
cs/11-email-api.md               <-- Email API reference
```

---

## Jak se orientovat

- **Pracujete na PgePilotu?** Zacnete s `01`, pak `06` (API), `07` (entity).
- **Pracujete na SmartBoxu?** Zacnete s `02`, pak `06` (RPC sekce), `07` (entity).
- **Pracujete na webove app?** Zacnete s `03`, pak `06` (API).
- **Potrebujete infrastrukturu?** Cti `05`.
- **Potrebujete velky obraz?** Cti `08` (architektura), pak `09` (roadmapa).
- **Bezpecnostni audit?** Cti `10`.

---

## Pristupy a hesla

Vsechna hesla v dokumentaci jsou oznacena `[REDACTED]`. Skutecne hodnoty jsou v:
- `zadani/pristupy a servery/PGEERP_Knowledge_Base.md` (privatni OneDrive)
- `zadani/pristupy a servery/pgepilot_server.md` (privatni OneDrive)

---

## Servery -- rychly prehled

| # | Server | IP | Ucel |
|---|--------|----|------|
| 1 | pgepilot.cz | 88.99.187.9 | PgePilot cloud (7 Docker kontejneru) |
| 2 | ra-energity.cz | 195.201.19.103 | SmartBox VPS (sb-manager, SSH tunely) |
| 3 | pgeusers | 188.245.255.117 | ERP aplikace (Pricelist, PMO, Auth, Admin, TaskHub) |
| 4 | pgen-data-analyse | 94.130.78.0 | EnerSim simulacni engine |

---

## Git repozitare

| Repo | Ucel |
|------|------|
| Holbytlo/pgepilot-service | PHP backend + API v2 |
| Holbytlo/pgepilot-js | Node.js job orchestrator |
| Holbytlo/pgepilot-srv | Auth server (JWT) |
| Holbytlo/pge-app | Vue3 frontend (PGE App) |
| Holbytlo/sb | SmartBox Python software |
| Holbytlo/sb-manager | SmartBox sprava zarizeni |
| Holbytlo/pgepilot-documentation | Tato dokumentace |

---

## Pravidla udrzby

1. **Anglicke docs v korenu jsou zdrojem pravdy.** Jsou psane AI-first (strukturovane, explicitni).
2. **Ceske docs v `cs/` jsou odvozene.** Po kazde zmene v korenu aktualizujte CZ verzi.
3. **Nikdy necommitujte credentials.** Pouzivejte `[REDACTED]` a odkazujte na privatni OneDrive.
4. **Udrzujte INDEX.md aktualni** pri pridani/odebrani souboru.
5. **Datujte kazdou aktualizaci** v hlavicce dokumentu (format: YYYY-MM-DD).
