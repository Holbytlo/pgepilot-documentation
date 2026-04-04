# 07 -- Entitni model (CZ)

> Zdrojovy dokument: [../07-entity-model.md](../07-entity-model.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-04

---

## Shrnuti

- **Novy model (pge_control)**: Control Domain -> Collection Point -> Device -> Machine.
- **Legacy model (pgepilot)**: User -> Customer -> Plant -> Machine.
- **Kriticke pravidlo**: legacy databaze se NIKDY nemodifikuji primo -- pouze cteni.
- **Backfill dokoncen**: data z legacy tabulek jsou prenesena do noveho modelu.
- **Severity 0-4**: klasifikace zavaznosti alertu a udalosti.

---

> Pro uplne detaily viz anglicka verze: [../07-entity-model.md](../07-entity-model.md)
