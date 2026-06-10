# Lokální WIP inventura nested checkoutů — 2026-06-10 (PGP-034)

Read-only audit 11 vnořených git repozitářů pod `~/projekty AI/pgepilot/` + pge-app worktrees.
Provedl claude_valVA_v2 (TaskAI PGP-034). Metodika: status/branch/ahead-behind, kategorizace
dirty souborů (feature/fix/scratch/generated), křížová kontrola obsahu proti všem remote
branchím (git cherry, ls-tree, grep přes refs/remotes), záchranné pushe bez dotyku working tree.

## Souhrn

| Repo | Checkout branch | Dirty | Nezachráněná práce | Akce |
|---|---|---|---|---|
| pge-app | codex/local-pge-app-wip-20260427 | 6 | ForecastAnalytics.vue +wiring (~1160 ř.); worktree app-modes: 4 commity PGP-046/047/049 + 19 dirty (PGP-048/050) | preserved → `codex/preserve-pge-app-forecast-analytics-20260610` (2d1f600), `codex/preserve-pge-app-app-modes-20260610` (7a115fc) |
| pgepilot-service | codex/local-pgepilot-service-wip-20260425 | 3 | commit ab3a937 = security fix PD-1055 (24 neautentizovaných debug GET rout vč. ovládání relé); migrations/012 (PD-606/607) jen lokálně | preserved → `codex/preserve-pd1055-debug-routes-fix-20260610` (ab3a937) |
| sb-manager | codex/local-sb-manager-wip-20260425 | 4 | sloupec „poznámka" v devices tabulce (~10 ř.); zbytek (SoLaX, CET) superseded na devva | preserved → `codex/preserve-sbmanager-wip-20260610` (8203532) |
| sb | codex/local-sb-wip-20260427 | 6 | žádná — vše obsahově na origin/devva, goodwe protection trojice = starší draft superseded 0b49804 | — |
| pge-auth | codex/pgepilot-auth-code-flow-bootstrap-20260419 | 0 | žádná | — |
| pge-mobile | codex/local-pge-mobile-wip-20260427 | 0 | žádná | — |
| pgepilot-documentation | codex/local-docs-wip-20260427 | 0 | žádná (lokál 7 za remote tipem) | — |
| pgepilot-js | codex/local-pgepilot-js-wip-20260427 | 0 | žádná | — |
| pgeworker-documentation | main | 0 | žádná | — |
| sb-router | main | 0 | žádná | — |
| servisdesk | codex/local-servisdesk-wip-20260427 | 0 | žádná | — |

## Klíčové nálezy

1. **PD-1055 security fix existoval jen na tomto disku.** Commit ab3a937 (odstranění 24
   neautentizovaných debug GET rout z Route_Pgep.php, vč. fyzického ovládání relé/střídače/baterie
   na hard-coded plant ID) nebyl na žádné remote branchi; origin/main, devva i reconcile-prod-20260503
   stále nesou zranitelnou 1477ř. verzi. Nezávisle přepsanou čistou verzi (495 ř.) má jen
   origin/server-dev. → zachráněno; follow-up: ověřit prod + integrovat (viz TaskAI follow-up task).
2. **pge-app dirty forecast-quality feature** (ForecastAnalytics.vue 955 ř. + api/router/App wiring)
   není na žádné z 27 remote branchí — pravděpodobně živý WIP běžící forecast v2 série
   (PGP-039/PGP-040, chatgpt in_progress). Zachráněno preservation branchí; neintegrovat
   mimo PGP-039.
3. **feature/app-modes (PGP-046/047/049) nebyla nikde pushnuta** + worktree nesl 19 necommitnutých
   souborů = deliverables PGP-048/050 popsané ve FINISH akcích, ale nikdy necommitnuté.
   Obojí zachráněno preservation branchí; publikace samotné feature/app-modes dál čeká na
   Vladimirovo OK (PGP-046).
4. **sb-manager „poznámka" sloupec** — jediný nezachráněný delta repa; SoLaX a CET části lokálního
   WIP jsou superseded novější implementací na origin/devva.
5. Scratch/generated: pgepilot-service migrations/011 (byte-identická kopie devva verze, smazatelná),
   routes_pge_control.php (starší iterace OTE/TOU, bohatší verze na devva), pge-app
   tsconfig.app.tsbuildinfo, smoke draft customer_demo_stable_privacy_smoke.sh (NOTE placeholdery).
6. Všechny checkouty jsou **stale pre-integrační snapshoty** (52–253 commitů za svými dev liniemi) —
   před další prací v nich vždy fetch/rebase; kandidát na úklidové kolo.

## Preservation branche (vše ověřeno ls-remote po pushi)

- Holbytlo/pgepilot-service `codex/preserve-pd1055-debug-routes-fix-20260610` = ab3a937
- Holbytlo/sb-manager `codex/preserve-sbmanager-wip-20260610` = 8203532
- Holbytlo/pge-app `codex/preserve-pge-app-forecast-analytics-20260610` = 2d1f600
- Holbytlo/pge-app `codex/preserve-pge-app-app-modes-20260610` = 7a115fc

Pushe proběhly přes temp-index commity (git commit-tree) — žádný working tree, index ani lokální
branch nebyly změněny.
