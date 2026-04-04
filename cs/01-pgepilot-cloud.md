# 01 -- PgePilot Cloud (CZ)

> Zdrojovy dokument: [../01-pgepilot-cloud.md](../01-pgepilot-cloud.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-04

---

## Shrnuti

- **Cloudova platforma pro monitoring FVE instalaci** -- 7 Docker kontejneru bezicich na pgepilot.cz (88.99.187.9).
- **23 aktivnich plantu**: 7 GoodWe, 15 SolaX, 1 SmartBox.
- **Task system**: JobManager -> Service -> Worker -- orchestrace ulohovych cyklu.
- **Predikcni system**: PV forecast (forecast.solar), load forecast, adaptivni korekce pro optimalizaci vykonu.
- **Konektorova self-governance**: kazdy konektor si ridi vlastni budget a cache.

---

> Pro uplne detaily viz anglicka verze: [../01-pgepilot-cloud.md](../01-pgepilot-cloud.md)
