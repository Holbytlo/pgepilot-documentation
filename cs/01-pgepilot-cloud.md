# 01 -- PgePilot Cloud (CZ)

> Zdrojovy dokument: [../01-pgepilot-cloud.md](../01-pgepilot-cloud.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-21

---

## Shrnuti

- **Cloudova platforma pro monitoring FVE instalaci** -- 7 Docker kontejneru bezicich na pgepilot.cz (88.99.187.9).
- **23 aktivnich plantu**: 7 GoodWe, 15 SolaX, 1 SmartBox.
- **Task system**: produkcni opakovane behy dnes ridi `task_definitions` v DB -> service `/run-tasks` -> JobManager -> worker.
- **Historicka data**: GoodWe backfill zapisuje kanonicke `power_1m`; klientsky profil `power_bf` se resolverem sklada nad touto historii a `energy_15m` nese lineage metadata.
- **Runtime poznamka**: `pgepilot_service` je aktualne na `da784a6`, zatimco workeri 1/2/3 zustavaji na `a04e0ba`; pri dalsim zasahu nepocitat s uplne stejnym SHA vsude.
- **Predikcni system**: forecast.solar a load forecast endpointy existuji, ale aktivni recurring flow v produkci dnes tvori hlavne Open-Meteo forecast (`26`) a forecast correction (`27`).
- **OTE SPOT import**: HTML scraping z `ote-cr.cz` (denni trh). Tasky `30` (00:05, dnesek) a `31` (12:10, dnes+zitra) ukladaji PT15M ceny do `pgepilot.ote_day_ahead_prices` (pozor: `pgepilot` DB, ne `pgepilot_service`). Overeno 2026-04-21: 15 dnu kontinualnich dat, 96 radku/den.
- **Konektorova self-governance**: kazdy konektor si ridi vlastni budget a cache.

---

> Pro uplne detaily viz anglicka verze: [../01-pgepilot-cloud.md](../01-pgepilot-cloud.md)
