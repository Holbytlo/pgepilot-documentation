# 05 -- Infrastruktura (CZ)

> Zdrojovy dokument: [../05-infrastructure.md](../05-infrastructure.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-12

---

## Shrnuti

- **3 servery Hetzner**: pgepilot.cz (7 Docker kontejneru), ra-energity.cz (SmartBox VPS), a dalsi.
- **8 databazi** na MariaDB pokryvajicich vsechny sluzby.
- **`pge_data`** dnes obsahuje kanonickou historii (`power_1m`, `power_rt`, `energy_15m`) a reportingove profily (`power_bf`, `power_1h`, `power_1d`), ktere se umi skladat nad kanonickou historii.
- **Deploy postupy** pro jednotlive kontejnery a servicy.
- **Zalohovani**: v soucasnosti neuplne -- identifikovano jako riziko.
- **Scheduler realita**: produkce uz neni popsatelna jen pres stary `/scheduled_tasks`; kanonicka pravda je dnes `task_definitions` + worker flow.
- **OTE import**: aktivni tasky `30` a `31` importuji PT15M ceny pro dnesek a zitrek.
- **Sitovy diagram** popisujici propojeni mezi servery, SmartBoxy a externimi sluzbami.

---

> Pro uplne detaily viz anglicka verze: [../05-infrastructure.md](../05-infrastructure.md)
