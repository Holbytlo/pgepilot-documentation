# 06 -- API Reference (CZ)

> Zdrojovy dokument: [../06-api-reference.md](../06-api-reference.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-12

---

## Shrnuti

- **API v2**: 36+ endpointu na /api/v2/* s JWT autentizaci.
- **Historie/reporting**: `/collection-points/{code}` vraci `history_usage_options`, `/history` a `/energy-summary` umi `usage` + `dataset`; `power_bf` je klientsky reporting profil, ktery se pri potrebe sklada z kanonickeho `power_1m`.
- **Legacy v1** stale bezi pro zpetnou kompatibilitu.
- **Email API**: POST /sendEmail pro odesilani emailu z aplikaci.
- **SmartBox RPC**: sb.* metody pro vzdalenou komunikaci se SmartBoxy.
- **OTE import**: korenovy endpoint `GET /ote/import` pro PT15M/PT60M denni trh OTE.
- **Task orchestrace**: aktualni produkce jede pres `task_definitions` -> `/run-tasks` -> JobManager `/add_task` -> worker `/task`.
- **Externi konektory**: GoodWe SEMS, SolaX Cloud, forecast.solar, Open-Meteo.

---

> Pro uplne detaily viz anglicka verze: [../06-api-reference.md](../06-api-reference.md)
