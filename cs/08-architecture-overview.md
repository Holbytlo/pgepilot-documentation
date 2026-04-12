# 08 -- Architektura - prehled (CZ)

> Zdrojovy dokument: [../08-architecture-overview.md](../08-architecture-overview.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-12

---

## Shrnuti

- **Celkovy pohled na system** PgePilot a vsechny jeho komponenty.
- **Datove toky** jsou ted popsane jako raw/source -> kanonicka historie -> reporting profil, plus usage-based resolver historie.
- **4 vyvojove faze**: A = SmartBox rizeni, B = nova DB (HOTOVO), C = API + klienti (C1 HOTOVO), D = pokrocile funkce.
- **Jedno API, tri klienti**: webova app, mobilni app, SmartBox -- vsichni sdili API v2.
- **Legacy koexistence**: stare i nove systemy bezi paralelne, legacy se postupne utlumuje.

---

> Pro uplne detaily viz anglicka verze: [../08-architecture-overview.md](../08-architecture-overview.md)
