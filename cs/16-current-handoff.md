# 16 -- Aktuální handoff (CZ)

> Krátký operační handoff pro další chat/okno.
> Poslední aktualizace: 2026-04-24

Tento soubor je záměrně krátký. Původní dlouhý handoff z 2026-04-16 byl přesunut do archivu:

`../../.archive/2026-04-24_docs_cleanup/pgepilot-documentation/cs-16-current-handoff.pre-cleanup-2026-04-24.md`

Nečíst archiv jako výchozí kontext. Otevřít ho jen při konkrétní historické diagnostice.

## Číst Nejdřív

1. Stav workspace: `../../docs/ACTIVE_STATUS.md`
2. Mapa dokumentace: `../INDEX.md`
3. Infrastruktura: `../05-infrastructure.md`
4. SmartBox/SB: `../02-smartbox-sbc.md`, `../14-smartbox-provisioning.md`, `../17-smartbox-new-image-checklist.md`
5. Auth: `../22-auth-schema-apps.md`, pak `../18-pgepilot-sluzby-prihlaseni.md`, pak `../19-pgepilot-auth-migration-plan.md`

## Aktuální Realita

- `lokál` je workspace s více git checkouty, ne jeden repozitář.
- V mnoha vnořených repozitářích jsou lokální WIP změny. Neresetovat.
- `server prod` je `pgepilot.cz`; read-only kontrola 2026-04-24 ukázala běžící produkční Docker kontejnery.
- `server dev` je `ra-energity.cz`; read-only kontrola 2026-04-24 ukázala běžící `sb-manager` a demo/simulator PM2 procesy.
- `GitHub upstream` jsou Holbytlo repozitáře. Některé fetch/push operace potřebují Holbytlo SSH klíč.

## Hranice Archivu

Staré handoffy, jednorázové plány, Superpowers dokumenty, MP135 runbooky a handoff balíky jsou v `.archive/2026-04-24_docs_cleanup/`.

AI agenti nemají číst `.archive/` bez explicitního požadavku na konkrétní historický artefakt.

