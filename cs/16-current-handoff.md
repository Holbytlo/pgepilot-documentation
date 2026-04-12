# 16 -- Aktualni Handoff

> Kratky operacni handoff pro dalsi chat/okno.
> Posledni aktualizace: 2026-04-12

---

## Strucny Stav

- `pgepilot_service`: `main@a921042`
- `pgepilot_worker1/2/3`: `main@a921042`
- `pgepilot_servicedesk:/home/app2/pge-app`: `main@3d7e6bb`
- `sb-manager`: `main@ef3261b`
- `task 18`, `task 19`, `task 22` a `task 23` jsou aktivni
- `task 20` a `task 21` zustavaji vypnute
- `task 18` je po oprave stabilni; v poslednich 6h melo `1205 completed / 0 failed`
- `task 19` je zatim taky zdravy; v poslednich 6h melo `1215 completed / 9 sent / 0 failed`

Dulezite:

- history cutover a auth/SmartBox helper zmena nebyly jedna vec
- service a workeri jsou ted zase na stejnem SHA
- SB1, SB4 a SB7 uz nesdili jednu cloud identitu
- otevreny problem uz neni shared identity, ale chybejici UUID validace pro `machine_id`

---

## Co Je Hotove

- kanonicka historie jde do `pge_data`
- GoodWe backfill zapisuje `power_1m`
- SmartBox `smartboxSendData` se mapuje do `power_1m` / `energy_15m`
- `task 22` zapisuje `realtime_state` do `power_1m`
- web uz pouziva novy `usage` kontrakt
- SB1, SB4 a SB7 maji oddelene `login + smartbox_id + collection_point_id + device_id`
- auditem overene klicove runtime soubory na SB1/SB4/SB7 odpovidaji lokalnimu `sb/devva@45a2fc0`

---

## Co Zbyva

- dal monitorovat `task 18` a `task 19`, ale aktualne nejsou zachycene nove faulty
- opravit a zvalidovat neplatne `machine_id`:
  - `sb1 relay`: `b1bb905b-d285-50e3-98df-g5e62g1gc645`
  - `sb7 inverter`: `c2cc916c-e396-51f4-a9f0-h6f73h2hd756`
- doplnit tvrdou UUID validaci do `sb-manager`
- `sb7` jeste nema finalni customer config: porad `enabled:false`, placeholder IP a placeholder `logger_serial`

---

## Kde Zacit

Cloud/history:

1. `05-infrastructure.md`
2. `06-api-reference.md`
3. `07-entity-model.md`
4. `08-architecture-overview.md`
5. `09-development-roadmap.md`
6. `15-sign-conventions.md`

SmartBox:

1. `02-smartbox-sbc.md`
2. `14-smartbox-provisioning.md`
3. `15-sign-conventions.md`

Klicove soubory:

- `pgepilot-service/src/PgePilot/Api/TaskController.php`
- `pgepilot-service/src/PgePilot/Api/ApiController.php`
- `pgepilot-service/src/PgePilot/DataAdmin/DataAdminPgep.php`
- `pgepilot-service/src/PgePilot/Services/SourcePolicyService.php`
- `pgepilot-service/src/PgePilot/Services/ConnectorPowerTransformService.php`
- `pgepilot-service/src/PgePilot/Services/PgeDataTimeSeriesStore.php`
- `sb/communication_controller/smartbox_service.py`
- `sb/rpc_client/models.py`

---

## Produkcni Kontroly

Stav tasku:

```bash
ssh root@pgepilot.cz \
  "mysql -uroot -pVeslo123 -N -e \
  \"SELECT id, name, active, last_run
     FROM pgep_tasks.task_definitions
    WHERE id IN (18,19,20,21,22,23)
    ORDER BY id;\""
```

Posledni failed backfill:

```bash
ssh root@pgepilot.cz \
  "mysql -uroot -pVeslo123 -N -e \
  \"SELECT id, status, generated_at, completed_at,
           LEFT(response, 200), LEFT(param_data, 300)
     FROM pgep_tasks.tasks
    WHERE task_definition_id = 18
      AND status = 'failed'
    ORDER BY id DESC
    LIMIT 10;\""
```

Logy:

```bash
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_worker1 2>&1'
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_worker2 2>&1'
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_worker3 2>&1'
ssh root@pgepilot.cz 'docker logs --tail 80 pgepilot_service 2>&1'
```
