# 01 -- PgePilot Cloud (CZ)

> Zdrojovy dokument: [../01-pgepilot-cloud.md](../01-pgepilot-cloud.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-07

---

## Shrnuti

- **Cloudova platforma pro monitoring FVE instalaci** -- 7 Docker kontejneru bezicich na pgepilot.cz (88.99.187.9).
- **23 aktivnich plantu**: 7 GoodWe, 15 SolaX, 1 SmartBox.
- **Task system**: produkcni opakovane behy dnes ridi `task_definitions` v DB -> service `/run-tasks` -> JobManager -> worker.
- **Predikcni system**: forecast.solar a load forecast endpointy existuji, ale aktivni recurring flow v produkci dnes tvori hlavne Open-Meteo forecast (`26`) a forecast correction (`27`).
- **OTE SPOT import**: PT15M importer bezi v produkci pres tasky `30` a `31` a uklada ceny do `ote_day_ahead_prices`.
- **Konektorova self-governance**: kazdy konektor si ridi vlastni budget a cache.

---

> Pro uplne detaily viz anglicka verze: [../01-pgepilot-cloud.md](../01-pgepilot-cloud.md)
