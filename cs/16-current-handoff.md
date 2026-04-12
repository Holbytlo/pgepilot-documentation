# 16 -- Aktualni Handoff

> Kratky operacni handoff pro dalsi chat/okno.
> Posledni aktualizace: 2026-04-12

---

## Strucny Stav

- `pgepilot_service`: `main@da784a6`
- `pgepilot_worker1/2/3`: `main@a04e0ba`
- `pgepilot_servicedesk:/home/app2/pge-app`: `main@3d7e6bb`
- `task 22` a `task 23` jsou aktivni
- `task 20` a `task 21` zustavaji vypnute
- `task 18` stale obcas pada na HTTP `500`

Dulezite:

- history cutover a auth/SmartBox helper zmena nebyly jedna vec
- service je ted o jeden commit pred workeri
- nepredpokladejte, ze service a workeri bezi na stejnem SHA

---

## Co Je Hotove

- kanonicka historie jde do `pge_data`
- GoodWe backfill zapisuje `power_1m`
- SmartBox `smartboxSendData` se mapuje do `power_1m` / `energy_15m`
- `task 22` zapisuje `realtime_state` do `power_1m`
- web uz pouziva novy `usage` kontrakt

---

## Co Zbyva

- dohledat, proc `task 18` stale obcas pada na `500`
- hlidat, ze service a workeri nejsou rozjeti vic nez dnes
- auth brat jako zvlastni thread, ne jako soucast history cutoveru

Jeden zachyceny fail:

```json
{
  "controllerFunction": "backfillHistoricalData",
  "params": {
    "plantId": "df559764-d5f3-4101-a3cb-cc1b7d1831f3",
    "dateFrom": "",
    "dateTo": "",
    "maxDays": 7
  }
}
```

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
ssh root@pgepilot.cz 'tail -n 80 /var/log/pgepilot/worker/taskcontroller.log'
ssh root@pgepilot.cz 'tail -n 80 /var/log/pgepilot/worker/taskcontroller.err'
```
