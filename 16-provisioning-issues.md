# 16 -- Provisioning Issues (zjisteno 2026-04-12)

> Problemy zjistene pri rucnim provisioningu sb7, sb13 a sb4.
> Kazdy problem ma popis, workaround, a navrh opravy.
> Posledni aktualizace: 2026-04-12

---

## Kriticke (blokuji automaticky provisioning)

### P1: sb_bootstrap.sh — HOST_APPLY_FAILED: SB_ID is required

**Kde:** VPS, `vps_register_tunnel_key.sh` volany z sb-manageru pri provisioningu
**Co se stalo:** Bootstrap na sb13 selhal 3x s `HTTP 500 HOST_APPLY_FAILED: SB_ID is required`
**Pricina:** sb-manager pri volani sudo skriptu nepredava SB_ID jako env promennou
**Workaround:** Rucni registrace klice v authorized_keys na VPS
**Oprava:** V `sb-manager/src/routes/provision.js` overit, ze `SB_ID` se predava do env pri `child_process.exec` volani `vps_register_tunnel_key.sh`
**Status (2026-04-12 vecer):** OPRAVENO v live `sb-manager` (`/opt/sb-manager/src/routes/provision.js` uz predava `SB_ID` do env)

---

### P2: sb-manager API neumi nastavit `n` pri vytvoreni zarizeni

**Kde:** `POST /api/v1/sb/devices`
**Co se stalo:** Vytvoreni sb13 priradilo `n=3` (prvni volny slot) misto `n=13`
**Pricina:** API automaticky vyplnuje nejnizsi volne `n`, neprijima ho jako parametr
**Workaround:** Rucni `UPDATE sb_devices SET n=13` primo v SQLite DB na VPS
**Oprava:** Pridat volitelny parametr `n` do POST `/api/v1/sb/devices` schema. Pokud neni zadan, pouzit auto-increment. Pokud je zadan, overit ze neni obsazeny.
**Status (2026-04-12 vecer):** OPRAVENO v live `sb-manager` API i UI (`createDeviceSchema` ma `n` a UI ma pole `n (volitelne)`)

---

### P3: sb-manager API nema DELETE endpoint

**Kde:** `DELETE /api/v1/sb/devices/:sbId` — 404 Not Found
**Co se stalo:** Nebylo mozne smazat spatne vytvorene zarizeni pres API
**Workaround:** Primo SQLite `DELETE FROM sb_devices WHERE sb_id=...`
**Oprava:** Pridat DELETE endpoint (soft-delete nebo hard-delete s potvrzenim)

---

### P4: sb-manager API nema PATCH pro `alias`

**Kde:** `PATCH /api/v1/sb/devices/:sbId` — `alias` je unrecognized key
**Co se stalo:** Sloupec `alias` existuje v DB ale neni v zod validacnim schematu
**Workaround:** Alias ulozeny v `notes` poli ("Listany KD (alias: listany_kd)")
**Oprava:** Pridat `alias` do PATCH schema v `src/routes/admin.js`

---

## Vysoke (zpusobuji manualni praci pri kazdem novem SB)

### P5: install_systemd_services.sh ma hardcoded cesty z SB1

**Kde:** `sb/systemd/install_systemd_services.sh`
**Co se stalo:** Na sb7 (CoreMP135) service unity meli cesty `/home/energity1/energity-development/` misto `/opt/energity/sb/`
**Pricina:** Skript a service soubory byly napsany pro SB1 (RPi, user energity1), ne pro obecny deploy
**Workaround:** Rucni vytvoreni 7 service unit souboru na kazdem novem zarizeni
**Oprava:** Bud parametrizovat cesty v install skriptu (detekce existujiciho adresare), nebo presunout service soubory do sablony ktera pouziva promenne (`$SB_ROOT`, `$VENV_PATH`)
**Status (2026-04-12 vecer):** OPRAVENO V REPU `sb/devva` — `sb/systemd/install_systemd_services.sh` ted renderuje templated unity s autodetekci `SB_ROOT`, user/group a venv. Pending deploy na dalsi boxy.

---

### P6: Python deps nejsou v requirements.txt

**Kde:** `sb/requirements.txt` obsahuje jen `PyYAML, requests, pymodbus, ntplib`
**Co se stalo:** Na sb7 a sb13 chybely `flask`, `werkzeug`, `fastapi`, `uvicorn`, `httpx`, `pydantic` — vsechny services crashovaly
**Pricina:** Kazda sluzba ma vlastni deps ktere nejsou v hlavnim requirements.txt
**Workaround:** Rucni `pip install flask werkzeug fastapi uvicorn httpx pydantic` po kazdm deploymentu
**Oprava:** Bud jeden spolecny `requirements.txt` se vsemi deps, nebo `requirements-all.txt` ktery je union vsech, nebo per-service requirements s install skriptem

---

### P7: python3-venv neni soucasti base image

**Kde:** Debian 12 (bookworm) na CoreMP135
**Co se stalo:** `python3 -m venv` selze s "ensurepip is not available"
**Workaround:** `apt install python3.11-venv` (nebo `python3-venv` na trixie)
**Oprava:** Pridat do bootstrap skriptu pred vytvorenim venv, nebo mit v base image

---

### P8: Watchdog neni soucasti produkcniho bootstrap skriptu

**Kde:** `sb-manager/src/templates/sb_bootstrap.sh`
**Co se stalo:** Funkce `enable_hardware_watchdog()` je pridana lokalne na Macu, ale produkci sb-manager na VPS ji jeste negeneruje
**Pricina:** Zmena v sablone nebyla deployovana na VPS
**Workaround:** Rucni `sed -i` na kazdem novem zarizeni
**Oprava:** Deploy aktualni `sb-manager` na VPS (commit `b48b296`)
**Status (2026-04-12 pozde vecer):** VYRESENO. Live `sb-manager` je ted `main@ef3261b` a aktualni `sb_bootstrap.sh` sablona obsahuje `enable_hardware_watchdog()` + `RuntimeWatchdogSec=30`.

---

### P9: PubkeyAuthentication je `no` na novem Debian image

**Kde:** `/etc/ssh/sshd_config` na cerstve flashnutem RPi OS / M5Stack Debian
**Co se stalo:** Root SSH klic nefungoval dokud jsme nezmenili `PubkeyAuthentication no` na `yes`
**Workaround:** Rucni sed + restart sshd
**Oprava:** Pridat do bootstrap skriptu: `sed -i 's/^PubkeyAuthentication no/PubkeyAuthentication yes/' /etc/ssh/sshd_config`

---

### P10: Node.js neni na cerstve Debian Trixie

**Kde:** RPi OS Lite (trixie) pro RPi 3
**Co se stalo:** sb-ops-agent (Node.js) nespustil — `node: command not found`
**Pricina:** Bootstrap skript ma `apt_install nodejs` ale ten se nespusti pokud bootstrap selze na provisioningu (P1)
**Workaround:** Rucni `apt install nodejs`
**Oprava:** Vyresit P1 (bootstrap dokonci vcetne node instalace)

---

## Stredni (neblokuji ale zpomaluji)

### P11: ServerAliveInterval=30 je moc pro LTE routery

**Kde:** `sb_bootstrap.sh` generuje ra-tunnel s `ServerAliveInterval=30`
**Co se stalo:** sb4 za LTE routerem (Mercusys MB112-4G) mel NAT timeout < 30s, tunel padal kazdych 30-60s
**Workaround:** Rucni zmena na `ServerAliveInterval=10` nebo `15`
**Oprava:** Bud snizit default na 15 v sablone, nebo pridat konfigurovatelny parametr `TUNNEL_KEEPALIVE_SEC` do bootstrap

---

### P12: StartLimitIntervalSec=300/Burst=10 je prilis restriktivni

**Kde:** ra-tunnel.service generovany bootstrapem
**Co se stalo:** Po pkill na VPS (cleanup zombie sessions) sb4 prestala restartovat tunel — dosahla 10 restartu za 5 min
**Workaround:** Rucni `StartLimitIntervalSec=0` na kazdem zarizeni
**Oprava:** Uz opraveno v lokalni sablone (`StartLimitIntervalSec=0`), ale neni deployovane na VPS

---

### P13: StartLimitIntervalSec nefunguje v [Service] sekci na Debian 12

**Kde:** systemd 252 (Debian 12 bookworm, CoreMP135)
**Co se stalo:** `Unknown key name 'StartLimitIntervalSec' in section 'Service', ignoring`
**Pricina:** V systemd < 254 musi byt `StartLimitIntervalSec` v `[Unit]`, ne v `[Service]`
**Workaround:** Presunout do `[Unit]` sekce
**Oprava:** Opravit v sablone bootstrap skriptu + v `install_systemd_services.sh`

---

### P14: VPS tcp_keepalive_time je defaultne 7200s (2 hodiny)

**Kde:** Linux kernel default na VPS ra-energity.cz
**Co se stalo:** CLOSE-WAIT TCP sockety z mrtvych tunelu blokovaly porty az 2 hodiny
**Workaround:** Uz opraveno: `sysctl net.ipv4.tcp_keepalive_time=30` (persistent v `/etc/sysctl.d/99-tunnel-keepalive.conf`)
**Status:** VYRESENO

---

### P15: YAML register map 37s parse time na ARM

**Kde:** `modbus_reg_goodwe.yaml` (2113 radku) na CoreMP135
**Co se stalo:** Device-controller startup trval 55s
**Workaround:** Uz opraveno: JSON cache (commit `0a0f747`), startup 4s
**Status:** VYRESENO

---

### P16: Chybi striktni UUID validace pro `machine_id`

**Kde:** `sb-manager/src/routes/admin.js`, sync/provisioning payloady pro cloud identity
**Co se stalo:** `machine_id` se validuje jen jako obecny string. Do live konfigurace a `pge_control` se tak propsaly i nevalidni hodnoty:

- `sb1 relay`: `b1bb905b-d285-50e3-98df-g5e62g1gc645`
- `sb7 inverter`: `c2cc916c-e396-51f4-a9f0-h6f73h2hd756`

**Dopad:** Chyba se neomezila jen na box configy; je uz i v `cp_connector_auth.machine_ids_json` a `cp_machines.id`.
**Workaround:** Rucne vygenerovat validni UUID a konzistentne je prepsat v cloudu i na boxu.
**Oprava:** Zprisnit validaci v `sb-manager` na UUID format pro `machine_id`, `masterMachineId`, `deviceId`, `collectionPointId`, `smartboxId` tam, kde to ma byt UUID, a pri sync/provisioningu failnout driv, nez se to zapise.
**Status (2026-04-12 pozde vecer):** Z VELKE CASTI VYRESENO. Live `sb-manager@21df16b` a `pgepilot-service@1727f60` uz UUID striktne validuji. Rozbita live ID na `sb1 relay` a `sb7 inverter` byla opravena end-to-end v cloudu, `sb-manager` SQLite i na boxech a sync byl znovu uspesne proveden. Primy live probe na `https://service.pgepilot.cz/api/v2/smartbox-auth/provision` mimo `sb-manager` vraci pro nevalidni `machine_id` korektne `HTTP 400` s chybou `machines[0].machine_id must be a valid UUID`. Samostatny follow-up zustava jen pro `sb13`, ktere se resi mimo tento provisioning wave.

---

## Navrhovany target stav (automaticky provisioning)

```
1. V sb-manageru vytvorit zarizeni s pozadovanym n
2. Stahnout bootstrap skript + ops-agent tarball
3. Flashnout SD kartu (RPi Imager)
4. Boot, pripojit Ethernet
5. Spustit bootstrap:
   - watchdog ON
   - python3-venv + vsechny deps
   - SSH key gen + registrace (bez HOST_APPLY bug)
   - ops-agent + nodejs
   - ra-tunnel s ServerAliveInterval=15 a StartLimitIntervalSec=0
   - PubkeyAuthentication=yes + root SSH klic
6. Rsync energity stack z Mac/git
7. Install systemd services (spravne cesty)
8. Hotovo — zarizeni online v sb-manageru
```

Krok 5 uz neni blokovany `P1`, ale stale obsahuje dalsi dily problemy (`P8`, `P9`, `P11`, `P12`).
Kroky 6-7 byly do 2026-04-12 vecer manualni kvuli `P5`; po fixu v repu uz chybi hlavne deploy na dalsi boxy a doostreni zbytku bootstrapu.
Cil: vsechno v jednom skriptu bez rucnich oprav po prvnim bootu.

---

## Priority oprav

| # | Effort | Impact | Popis |
|---|--------|--------|-------|
| P1 | Small | KRITICKE | Fix SB_ID env v provision.js -- HOTOVO v live sb-manageru |
| P2 | Small | VYSOKE | Pridat volitelny `n` parametr do POST devices -- HOTOVO v live API/UI |
| P5 | Medium | VYSOKE | Parametrizovat cesty v service unitech -- HOTOVO v repu `sb/devva`, pending deploy |
| P6 | Small | VYSOKE | Kompletni requirements.txt |
| P8 | ~~Small~~ | ~~VYSOKE~~ | ~~Deploy aktualni sb-manager na VPS~~ -- HOTOVO, live je `ef3261b` |
| P9 | Small | VYSOKE | PubkeyAuthentication fix v bootstrap |
| P3 | Small | STREDNI | DELETE endpoint pro devices |
| P4 | Small | STREDNI | Alias v PATCH schema |
| P11 | Small | STREDNI | ServerAliveInterval default 15 |
| P12 | Small | STREDNI | StartLimitIntervalSec=0 default |
| P13 | Small | STREDNI | StartLimit do [Unit] sekce |
| P16 | ~~Small~~ | ~~VYSOKE~~ | ~~UUID validace pro `machine_id` a oprava uz zapsanych spatnych ID~~ -- HOTOVO pro SB1/SB7, otevreny zbyva uz jen samostatny pripad SB13 |
