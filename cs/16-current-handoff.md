# 16 -- Aktuální handoff (CZ)

> Krátký operační handoff pro další chat/okno.
> Poslední aktualizace: 2026-05-03

Tento soubor je záměrně krátký. Původní dlouhý handoff z 2026-04-16 byl přesunut do archivu:

`../../.archive/2026-04-24_docs_cleanup/pgepilot-documentation/cs-16-current-handoff.pre-cleanup-2026-04-24.md`

Nečíst archiv jako výchozí kontext. Otevřít ho jen při konkrétní historické diagnostice.

## Číst Nejdřív

1. Stav workspace: `../../docs/ACTIVE_STATUS.md`
2. Mapa dokumentace: `../INDEX.md`
3. Infrastruktura: `../05-infrastructure.md`
4. SmartBox/SB: `../02-smartbox-sbc.md`, `../14-smartbox-provisioning.md`, `../17-smartbox-new-image-checklist.md`
5. Auth: `../22-auth-schema-apps.md`, pak `../18-pgepilot-sluzby-prihlaseni.md`, pak `../19-pgepilot-auth-migration-plan.md`
6. Operační/project KB: `/Users/vladimiradam/projekty AI/pge-knowledgebase/AI_INDEX.md`

## Aktuální Kritický Handoff

Nejnovější live zjištění ke SmartBox/GoodWe je v operační KB:

- `/Users/vladimiradam/projekty AI/pge-knowledgebase/projects/pgepilot/handovers/2026-05-03-planicka-goodwe-parallel-charging.md`
- `/Users/vladimiradam/projekty AI/pge-knowledgebase/projects/pgepilot/sites/planicka-sb6.md`
- `/Users/vladimiradam/projekty AI/pge-knowledgebase/projects/pgepilot/inverters/goodwe/parallel-goodwe-et.md`

Shrnutí:

- Plánička/SB6 je 2x GoodWe ET paralelně.
- Oba dongle IP přístupy ukazují stejná Modbus unit ID: `247` master/system a `1` slave.
- Slave `unit 1` vrací `EXC:2` pro běžné ARM řídicí registry (`47000/47120`, ECO/TOU sloty, force-charge, peak-shaving, smart/parallel bloky).
- Live testy 2026-05-03 nenašly bezpečnou ne-EMS Modbus cestu pro nabíjení ze sítě.
- `47511/47512` nebyly zapsané a zůstávají blokované bez explicitního servisního test plánu.
- Pro `goodwe_parallel_et` nesmí automatické OTE/spot řízení používat nenulové `47120` ani automatické grid charging. Do ověření dedikované strategie používat jen export-limit/stop-export cestu.

## Aktuální Realita

- `lokál` je workspace s více git checkouty, ne jeden repozitář.
- V mnoha vnořených repozitářích jsou lokální WIP změny. Neresetovat.
- `server prod` je `pgepilot.cz`; cílená read-only kontrola 2026-05-03 ukázala runtime artefakty bez viditelného `.git` v kontrolovaných kontejnerech a veřejné health/app/demo URL vracely `HTTP 200`.
- `server dev` má dvě relevantní plochy: `pgepilot-dev.cz` / `37.27.32.17` a SmartBox/VPS povrch na `ra-energity.cz`. Ověřit konkrétní cíl podle tasku.
- `ra-energity.cz:/opt/sb-manager` dirty runtime změny jsou zachované na GitHub branchi `codex/reconcile-sb-manager-server-dev-20260503` (commit `f084846`), bez deploye nebo merge.
- SB demo runtime checkouty pod `/home/limited/pgepilot-sb*-demo/sb` zůstávají dirty a potřebují samostatné rozdělení config/code/AppleDouble šumu.
- `GitHub upstream` jsou Holbytlo repozitáře. Některé fetch/push operace potřebují Holbytlo SSH klíč.

## Aktuální TaskAI Směrování

- PGP-051 je živý task pro SB1/SB6 sjednocení runtime změn a GoodWe protection strategii.
- PGP-046 až PGP-050 jsou PGE App safety/app-mode tasky čekající na Vladimirovu verifikaci.
- PGP-038 až PGP-044 jsou Forecast v2 tasky v progresu.
- PGP-001 a PGP-003 čekají na verifikaci; PGP-002 zůstává blokované, dokud nebude dořešené PGP-001 a GoodWe parallel strategie.

## Hranice Archivu

Staré handoffy, jednorázové plány, Superpowers dokumenty, MP135 runbooky, handoff balíky,
rozbité demo worktree pointery, lokální cleanup artefakty a široké runtime snapshoty jsou
v `.archive/` cold storage.

AI agenti nemají číst `.archive/` bez explicitního požadavku na konkrétní historický artefakt.
