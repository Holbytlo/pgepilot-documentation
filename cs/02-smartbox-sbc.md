# 02 -- SmartBox SBC (CZ)

> Zdrojovy dokument: [../02-smartbox-sbc.md](../02-smartbox-sbc.md) (anglicky, AI-first)
> Posledni aktualizace: 2026-04-04

---

## Shrnuti

- **SmartBox edge system** -- Raspberry Pi instalovany primo u zakaznika pro lokalni rizeni.
- **VPS ra-energity.cz** (195.201.19.103) slouzi ke sprave a vzdalenemu pristupu.
- **SB1**: RPi 4, Debian 13, pripojeny ke GoodWe ET+ invertoru pres Modbus TCP.
- **4 mikroservicy**: Device Controller, Local DB, RPC Client, Communication Controller.
- **TOU engine** pro casove rizeni tarifu a reverzni SSH tunel pro vzdalenou spravu.

---

> Pro uplne detaily viz anglicka verze: [../02-smartbox-sbc.md](../02-smartbox-sbc.md)
