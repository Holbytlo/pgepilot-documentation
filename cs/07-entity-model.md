# 07 -- Entitni model (CZ)

> Zdrojovy dokument: [../07-entity-model.md](../07-entity-model.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-12

---

## Shrnuti

- **Novy model (pge_control)**: Control Domain -> Collection Point -> Device -> Machine.
- **Legacy model (pgepilot)**: User -> Customer -> Plant -> Machine.
- **Kriticke pravidlo**: legacy databaze se NIKDY nemodifikuji primo -- pouze cteni.
- **`pge_data`** drzi kanonickou historii i reportingove profily; vyber pro API neridi jen nazev tabulky, ale `dataset_registry` + `data_source_registry`, a `power_bf` muze byt i logicky profil nad `power_1m`.
- **Severity 0-4**: klasifikace zavaznosti alertu a udalosti.

---

> Pro uplne detaily viz anglicka verze: [../07-entity-model.md](../07-entity-model.md)
