# 14 -- SmartBox Provisioning (kompletni postup)

> Kompletni navod na instalaci noveho SmartBox zarizeni od nuly az po produkci.
> Posledni aktualizace: 2026-04-14

---

## Prehled

Kazdy novy SmartBox (SB) projde temito fazemi:

```
1. Registrace v sb-manageru (web UI)
2. Flash SD karty (Debian image)
3. Prvni boot + sit (WiFi nebo Ethernet)
4. Bootstrap skript (sb_bootstrap.sh) — automatizovano
5. Instalace aplikacniho stacku (energity sb/)
6. Konfigurace zarizeni (stridac, cloud, identita)
7. Overeni na VPS (sb-manager health check)
8. Predani zakaznikovi
```

---

## Faze 1: Registrace v sb-manageru

Na sb-manazer.ra-energity.cz:

1. **Pridat zarizeni** — vyplnit device label (napr. `sb7`), poznamky
2. sb-manager priradi:
   - `sbId` (unikatni identifikator, napr. `sb_01KNW14X98...`)
   - `n` (poradove cislo zarizeni, urcuje base port)
   - base port = `20000 + (n-1) * 10`
3. **Vygenerovat provisioning token** (jednorazovy, platnost 1h):
   ```bash
   curl -sS -X POST \
     -H "X-Dev-User: ops@local" \
     -H "X-Dev-Groups: pge_sbmanager_ops" \
     -H "Content-Type: application/json" \
     -d '{"purpose":"REGISTER_TUNNEL_KEY","ttlSeconds":3600}' \
     "https://sb-manazer.ra-energity.cz/api/v1/sb/devices/<SB_ID>/provision-tokens"
   ```
4. **Stahnout bootstrap skript** (obsahuje token, port plan, VPS adresu):
   ```bash
   curl -sS -X POST \
     -H "X-Dev-User: ops@local" \
     -H "X-Dev-Groups: pge_sbmanager_admin" \
     "https://sb-manazer.ra-energity.cz/api/v1/sb/devices/<SB_ID>/bootstrap"
   ```
5. **Stahnout sb-ops-agent tarball** ze sb-manageru (sekce "Baliky")

---

## Faze 2: Flash SD karty

Kanonicky operativni checklist je ted v `17-smartbox-new-image-checklist.md`.
Historicky MP135-only runbook v `mp135/debian-flash-runbook.md` ber jen jako pomocny lokalni artifact, ne jako hlavni source of truth.

### Strucne:
1. Extrahovat `.7z` image (M5Stack Debian 12 pro CoreMP135, nebo standardni RPi OS pro Raspberry Pi)
2. Flashnout pres **Raspberry Pi Imager** → "Use custom" → vybrat `.img` → vybrat SD kartu → Write
3. Po "Write Successful" vlozit kartu do zarizeni

### Podporovane platformy:

| Platforma | OS | Architektura | Image |
|-----------|-----|-------------|-------|
| Raspberry Pi 4 | Debian 13 (trixie) | aarch64 (arm64) | Standardni RPi OS |
| M5Stack CoreMP135 | Debian 12 (bookworm) | armv7l | `M5_CoreMP135_debian12_20240919.7z` |

---

## Faze 3: Prvni boot + sit

1. **Pripojit napajeni** — prvni boot trva dele (partition expansion, cloudinit)
2. **Prihlasit se** pres serialovou konzoli:
   ```bash
   screen /dev/cu.usbmodem101 115200
   # Login: root / root (M5Stack) nebo pi / raspberry (RPi)
   ```
3. **Pripojit WiFi** (pokud neni Ethernet):
   ```bash
   # Pripojit:
   nmcli device wifi connect "SSID" password "heslo"

   # Overit:
   ping -c 2 ra-energity.cz
   ```
   Poznamka:
   Na current MP135 image nepredpokladej, ze libovolny USB Wi-Fi dongle bude fungovat. Napr. `RT5370` (`148f:5370`) na aktualnim kernelu nevytvori `wlan0`, protoze kernel nema zapnute `CONFIG_RT2800USB_RT53XX`. Viz `17-smartbox-new-image-checklist.md`.
4. **Nastavit hostname** (volitelne):
   ```bash
   hostnamectl set-hostname sb7
   ```

---

## Faze 4: Bootstrap skript (sb_bootstrap.sh)

Toto je hlavni automatizovany krok. Skript dela vse potrebne:

### Co skript automaticky udela:

| Krok | Co | Detail |
|------|-----|--------|
| 1 | Zkontroluje misto na disku | Pokud < 500 MB, pokusi se expandovat partition (growpart + resize2fs) |
| 2 | Nainstaluje prerekvizity | `apt-get install curl jq openssh-client ca-certificates` |
| 3 | **Zapne hardware watchdog** | Cílový stav: `RuntimeWatchdogSec=30` v `/etc/systemd/system.conf` + `daemon-reexec`. Produkční `sb-manager` šablona to k 2026-04-12 ještě negeneruje automaticky. |
| 4 | Vytvori uzivatele `energity` | Systemovy ucet pro tunel a sluzby |
| 5 | Vygeneruje SSH klic | ed25519 do `/home/energity/.ssh/tunnel_vps` |
| 6 | Zaregistruje klic na sb-manageru | POST `/api/v1/provision/register` s jednorazovym tokenem |
| 7 | Nainstaluje sb-ops-agent | Rozbaleni tarballu + systemd service na portu 3002 |
| 8 | Nainstaluje ra-tunnel.service | Reverzni SSH tunel na VPS (5 port forwardu) |
| 9 | Zapise status | `/var/lib/energity/install/status.json` s fazemi |

### Spusteni:

```bash
# Zkopirovat na zarizeni (USB, scp, nebo paste):
mkdir -p /home/energity/bootstrap
# ... sem nakopirovat sb_bootstrap_<label>.sh + sb-ops-agent.tar.gz

# Spustit:
sudo bash /home/energity/bootstrap/sb_bootstrap_<label>.sh
```

> **Reality check (2026-04-12):** Produkcni `sb-manager` na VPS zatim jeste negeneruje bootstrap se zapnutim watchdogem. Tato dokumentace popisuje cilovy postup a lokalni WIP v repu. Na novem zarizeni je proto stale potreba watchdog po bootstrapu overit a pripadne dodelat rucne.

### Jak overit ze probehl spravne:

```bash
# Watchdog zapnuty?
systemctl show -p RuntimeWatchdogUSec --value
# Ocekavano: 30s (nebo 1min na RPi — HW minimum)

# Tunel bezi?
systemctl is-active ra-tunnel
# Ocekavano: active

# sb-ops-agent bezi?
systemctl is-active sb-ops-agent
# Ocekavano: active

# Health endpoint odpovida?
curl -s http://127.0.0.1:3002/health | python3 -c "import sys,json; d=json.load(sys.stdin); print('ok:', d['ok'])"
# Ocekavano: ok: True

# Status bootstrapu?
cat /var/lib/energity/install/status.json
# Ocekavano: {"stage":"DONE","message":"Bootstrap finished",...}
```

---

## Faze 5: Instalace aplikacniho stacku

Bootstrap nainstaluje jen infrastrukturu (tunel, ops-agent, watchdog). Aplikacni kod (device-controller, local-db, atd.) se nasazuje zvlast.

### Postup (rsync z Macu):

```bash
# 1. Overit ze zarizeni je dostupne pres tunel:
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 "hostname; uptime"

# 2. Rsync kodu z Mac repa (branch devva):
cd ~/projekty\ AI/pgepilot/sb
git checkout devva

rsync -az --delete \
  -e 'ssh -J limited@ra-energity.cz -p <SSH_PORT>' \
  --exclude='.git/' --exclude='.venv/' --exclude='__pycache__/' \
  --exclude='*.pyc' --exclude='.DS_Store' --exclude='._*' \
  --exclude='local_database/*.db*' --exclude='local_database/*.sqlite*' \
  --exclude='logs/' --exclude='data/' \
  ./ root@127.0.0.1:/opt/energity/sb/

# 3. Na zarizeni: vytvorit venv a nainstalovat deps
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 "
  cd /opt/energity/sb
  python3 -m venv .venv
  ./.venv/bin/pip install --quiet -r requirements.txt
  chown -R energity:energity /opt/energity/sb/
  chown -R root:root /opt/energity/sb/.venv/
"

# 4. Nainstalovat systemd services:
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 "
  bash /opt/energity/sb/systemd/install_systemd_services.sh
"
```

### SSH porty podle zarizeni:

| Zarizeni | n | Base port | SSH port |
|----------|---|-----------|----------|
| sb1 | 1 | 20000 | 20002 |
| sb4 | 4 | 20030 | 20032 |
| sb5 | 5 | 20040 | 20042 |
| sb6 | 6 | 20050 | 20052 |
| sb7 | 7 | 20060 | 20062 |

### 5b) LCD dashboard (autodetekce podle realneho HW)

Pro vsechny podporovane platformy se ma pouzivat jeden dashboard runtime na boxu:

- `/opt/energity/sb-ops-agent/display_dashboard.py`
- `energity-display.service`

Pravidlo:

- provisioning pripravi kernel overlay a balicky pro konkretni fyzicky displej
- samotny dashboard si pri startu sam najde framebuffer, rozliseni a touch input, ktere jsou na stroji skutecne pritomne
- v aplikaci se nesmi vybirat displej podle nazvu boxu typu `sb4` nebo `sb13`

**CoreMP135 / M5Stack (`sb4`, `sb7`)**

- vestaveny displej ILI9342C, `320x240`
- framebuffer typicky `/dev/fb1`
- touch pres `evdev`
- neni potreba extra konfigurace v aplikaci

**Raspberry Pi 3 + 3.5" TFT HAT (`sb13` / Listany KD class)**

- ILI9486, `480x320`
- framebuffer muze byt `/dev/fb0` nebo `/dev/fb1` podle rotace
- touch primarne pres SPI polling, fallback `evdev`
- nutny overlay `tft35a` z `goodtft/LCD-show`

Minimalni kroky na RPi displeji:

```bash
# 1. Zkopirovat overlay
cp LCD-show/usr/tft35a-overlay.dtb /boot/firmware/overlays/tft35a.dtbo

# 2. Pridat do /boot/firmware/config.txt
dtparam=spi=on
dtoverlay=tft35a:rotate=90

# 3. Po rebootu nainstalovat deps
apt install -y fonts-dejavu-core python3-dev libopenjp2-7
/opt/energity/sb/.venv/bin/pip install Pillow spidev RPi.GPIO evdev
```

Poznamky:

- nepouzivat `piscreen` ani `fbtft` overlay na kernelu 6.12; crashuji na GPIO konfliktu
- pokud dashboard nenajde podporovany displej, ma skoncit ciste a nesmi rozbit ostatni SmartBox sluzby
- navigace ma byt jednotna: dole tecky + viditelne sipky `<` a `>` po stranach

---

## Faze 6: Konfigurace zarizeni

> Od 2026-04-12 je kanonicka SmartBox identita generovana z `sb-manager -> pgepilot-service -> cp_*`.
> Repo soubory `rpc_client_config.yaml`, `comm_controller_config.yaml` a `smartbox_config.yaml` jsou uz jen sablony.
> Na novem boxu se nesmi nechat vychozi hodnoty z repa; bez vygenerovane identity bundle ma service radeji failnout, nez se hlasit pod cizim boxem.

### Kanonicky identity bundle

Kazdy SmartBox musi dostat vlastni:

- `sb_id` — fyzicky box v `sb-manager`
- `collection_point_id` — lokalita v `cp_collection_points`
- `device_id` — SmartBox device v `cp_devices`
- `smartbox_id` — auth identita v `cp_connector_auth`
- `machine_ids[]` — stroje v `cp_machines`

Plati:

- `login` je jen credential
- `smartbox_id` je identita boxu
- `machine_id` je identita konkretniho stroje
- `plantId` se pro SmartBox provisioning uz nepouziva jako zdroj pravdy

### 6a) Konfigurace stridace (devices_config.yaml)

Na zarizeni editovat `/opt/energity/sb/device_controller/devices_config.yaml`:

**GoodWe stridac (Modbus TCP primo):**
```yaml
devices:
- name: inverter
  pretty_name: GoodWe ET+
  device_type: Inverter
  machine_id: <UNIKATNI_UUID>
  driver_type: modbus
  device_driver: GoodWeInverterDriver
  communication_driver: ModbusTCPDriver
  registers_config: drivers/devices/inverter_modbus_registers/modbus_reg_goodwe.yaml
  connection:
    mode: static
    host: <IP_STRIDACE>    # napr. 192.168.0.70
    port: 502
    unit_id: 247            # GoodWe default
  polling_interval: 5
  enabled: true
```

**GoodWe autodetekce v siti (kanonicky commissioning default pro nove boxy):**
```yaml
devices:
- name: inverter
  pretty_name: GoodWe ET+
  device_type: Inverter
  machine_id: <UNIKATNI_UUID>
  driver_type: modbus
  device_driver: GoodWeInverterDriver
  communication_driver: ModbusTCPDriver
  registers_config: drivers/devices/inverter_modbus_registers/modbus_reg_goodwe.yaml
  connection:
    mode: dhcp
    port: 502
    unit_id: 247
    discovery_cache_ttl: 300
  polling_interval: 5
  enabled: true
```

Pravidla pro GoodWe autodetekci:
- funguje jen pro `GoodWeInverterDriver`
- kdyz je `mode: dhcp` a **neni** zadana `mac_address`, SmartBox aktivne skenuje lokalni subnet a zkousi GoodWe Modbus registry
- pokud najde presne jeden kandidat, pouzije ho
- pokud najde vic kandidatu, failne zamerne jako `ambiguous`; v tom pripade je potreba konfiguraci zpresnit na `static host` nebo `dhcp + mac_address`
- `dhcp + mac_address` zustava podporovany a je vhodny tam, kde se IP meni, ale zarizeni musi byt jednoznacne svazane s jednim kusem HW

**Kanonicky workflow v `sb-manageru`:**
- device detail ma mit nastavovani **primarniho konektoru** (typ, `enabled`, `pretty_name`, `machine_id`, connection mode)
- pro GoodWe musi byt v `sb-manageru` dostupne tlacitko `Detekovat v siti`
- `sb-manager` zapisuje box-side `devices_config.yaml` pres SB `config-api` a po ulozeni restartuje `energity-device-controller`
- build scripts pro nove SD image maji pro `GoodWe` generovat vychozi `dhcp`/auto-scan konfiguraci, ne placeholder statickou IP

**Deye stridac (Modbus TCP primo):**
```yaml
devices:
- name: inverter
  pretty_name: Deye SUN-xK-SG04LP3-EU
  device_type: Inverter
  machine_id: <UNIKATNI_UUID>
  driver_type: modbus
  device_driver: DeyeInverterDriver
  communication_driver: ModbusTCPDriver
  registers_config: drivers/devices/inverter_modbus_registers/modbus_reg_deye.yaml
  connection:
    mode: static
    host: <IP_STRIDACE>    # napr. 192.168.1.100
    port: 502
    unit_id: 1              # Deye default (NE 247 jako GoodWe!)
  polling_interval: 5
  enabled: true
```

**Deye pres Solarman dongle (LSW-3, port 8899):**
```yaml
  communication_driver: SolarmanDriver
  connection:
    host: <IP_DONGLU>
    port: 8899              # Solarman V5 port
    unit_id: 1
    logger_serial: <SERIAL_Z_DONGLU>  # napr. 2712345678
```

### 6b) Cloud credentials (rpc_client_config.yaml)

Editovat `/opt/energity/sb/rpc_client/rpc_client_config.yaml`:
```yaml
pgepilot:
  auth_url: "https://service.pgepilot.cz"   # current default for new/migrated boxes
  rpc_url: "https://service.pgepilot.cz/rpc"

credentials:
  username: "sbx_<label>"   # napr. sbx_demo_box
  password: "<heslo>"

identifiers:
  smartbox_id: "<uuid>"
  collection_point_id: "<uuid>"
  device_id: "<uuid>"
  master_machine_id: "<uuid>"
```

Auth rollout rule (verified 2026-04-12):
- default for new provisioning is `https://service.pgepilot.cz`
- `https://auth.pgepilot.cz` remains only as legacy compatibility path for older boxes that are not yet migrated
- SB1, SB4, and SB7 already use `https://service.pgepilot.cz`

Current caveat:
- every SmartBox must keep its own connector/auth identity in `pge_control.cp_connector_auth`
- old shared row `sbx_deye25` may remain in DB as legacy residue, but active boxes must not use it

### 6c) Identita SmartBoxu (smartbox_config.yaml)

Editovat `/opt/energity/sb/communication_controller/smartbox_config.yaml`:

- `smartbox.id`
- `smartbox.collection_point_id`
- `smartbox.device_id`
- `machines[].machine_id`
- `device_to_machine`

### 6d) Restart sluzeb po konfiguraci:

```bash
systemctl restart energity-device-controller energity-comm-controller \
  energity-rpc-client energity-config-api energity-web-interface \
  energity-local-db energity-logs-db
```

---

## Faze 7: Overeni na VPS

### Z Macu nebo VPS:

```bash
# 1. Je zarizeni online v sb-manageru?
# → otevrit sb-manazer.ra-energity.cz, zkontrolovat zeleny badge "online"

# 2. Health check pres tunel:
ssh limited@ra-energity.cz "curl -s http://127.0.0.1:<BASE+5>/health" | python3 -m json.tool

# Ocekavano:
# {
#   "ok": true,
#   "data": {
#     "internal": { "allUp": true, "up": 6, "total": 6, "services": [...] },
#     "system": {
#       "watchdog": { "enabled": true, "timeout": "30s" },
#       "uptimeSeconds": ...,
#       "kernel": "...",
#       "diskFreeBytes": ...
#     }
#   }
# }

# 3. Device-controller log (prvni poll cyklus):
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 \
  "journalctl -u energity-device-controller --no-pager -n 20"

# Ocekavano:
#   "Service initialized with 1 devices"
#   "Connected to <IP>:502"
#   WARNING: Invalid scale factor ... (pre-existing, ne blocker)

# 4. sensor_data.db roste?
ssh -J limited@ra-energity.cz -p <SSH_PORT> root@127.0.0.1 \
  "stat -c '%y  size=%s' /opt/energity/sb/local_database/sensor_data.db"
# Kontrolovat ze mtime je cerstvy a size roste
```

### Compliance checklist (sb-manager kontroluje automaticky kazdych 30s):

| Kontrola | Zdroj | Warning pri selhani |
|----------|-------|---------------------|
| Zarizeni online | TCP probe na ops-agent :3002 | device offline v UI |
| Vsechny sluzby up | ops-agent /health → allUp | sluzba dole v UI |
| Watchdog zapnuty | ops-agent /health → system.watchdog.enabled | `WATCHDOG_OFF` v logu sb-manageru |
| Dostatek mista | ops-agent /health → system.diskFreeBytes | `LOW_DISK` v logu sb-manageru (< 500 MB) |

---

## Faze 8: Predani zakaznikovi

### Checklist pred odjezdem:

- [ ] sb-manager ukazuje zarizeni jako **online** se zelenym badge
- [ ] Health: **allUp=true, 6/6 services**
- [ ] Watchdog: **enabled=true** (v /health response)
- [ ] Device-controller: **1 device initialized** (v journalctl)
- [ ] sensor_data.db: **roste** (data z inverteru tece)
- [ ] rpc-client: **login 200 OK** proti nakonfigurovanemu auth endpointu (`service.pgepilot.cz` pro nove/migrovane boxy, `auth.pgepilot.cz` jen pro legacy boxy)
- [ ] ra-tunnel: **active** (zarizeni bude dostupne vzdalene)
- [ ] WiFi/Ethernet: **stabilni** (ping na ra-energity.cz funguje)

### Co delat na miste u zakaznika:

1. Pripojit SB do zakaznikova routeru (Ethernet preferovany, WiFi jako fallback)
2. Overit ze SB ma pristup ven (port 22 na 195.201.19.103 — nekteri routery blokuji)
3. Zjistit IP stridace v lokalni siti (napr. 192.168.1.x — z displeje stridace nebo routeru)
4. Editovat `devices_config.yaml` s realnou IP
5. `systemctl restart energity-device-controller`
6. Sledovat `journalctl -fu energity-device-controller` — prvni uspesny poll = hotovo
7. Overit ze cisla davaji smysl (vykon PV, SOC baterie, export) vs. displej stridace

### Fallback (pokud neco nefunguje):

| Problem | Reseni |
|---------|--------|
| Tunel se nepripoji | Zakaznikuv router blokuje port 22 ven. Otevrit, nebo pouzit SIM modem. |
| Stridac neodpovida | Spatna IP, spatny port (502 vs 8899), spatny unit_id (247 vs 1). |
| Data jsou nesmyslna | Spatna register mapa. Ladit `modbus_reg_*.yaml`, ne Python kod. |
| Zarizeni zamrzlo | Watchdog ho restartuje do 30s. Pokud ne — power cycle. |
| VPS neviditelny | Stary tunel zombie na VPS. `sudo pkill -u ra_tunnel` na VPS, cekat 90s. |

---

## Dulezita pravidla

1. **Jeden stridac = jeden SmartBox.** Dva SmartBoxy na jednom stridaci se perou o Modbus TCP socket a oba prestane cist. (Overeno 2026-04-12 na GoodWe.)

2. **Watchdog musi byt zapnuty na kazdem zarizeni.** Bez nej zamrzly SB ceka na fyzicky power-cycle. Bootstrap to dela automaticky od 2026-04-12.

3. **Per-machine config soubory NIKDY neprespisovat pri rsync deployi:**
   - `devices_config.yaml` (stridac IP, enabled stav)
   - `comm_controller_config.yaml` (collection_point_id/device_id pro comm-controller)
   - `rpc_client_config.yaml` (cloud credentials)
   - `smartbox_config.yaml` (identita SmartBoxu)

4. **Dashboard musi autodetekovat lokalni displej.** Provisioning pripravi HW a balicky, ale aplikace nesmi mit per-box hardcoded volbu typu displeje.

5. **VPS sshd ma ClientAliveInterval=30** — mrtve tunely se automaticky uklidi do 90s. Netreba rucne cistit (od 2026-04-12).

---

## Odkazy na souvisejici dokumenty

| Dokument | Cesta | Obsah |
|----------|-------|-------|
| SmartBox architektura | `pgepilot-documentation/02-smartbox-sbc.md` | SB1/sb4 detail, sluzby, driver hierarchy, known issues |
| Infrastruktura | `pgepilot-documentation/05-infrastructure.md` | VPS, deploy procedure, servery |
| Development roadmap | `pgepilot-documentation/09-development-roadmap.md` | Handoff items, hardening tasks |
| Flash runbook (CoreMP135) | `mp135/debian-flash-runbook.md` | SD karta, image, RPi Imager |
| sb4 handover | `mp135/HANDOVER-sb4.md` | Kompletni sb4 device state |
| Bootstrap skript (sablona) | `sb-manager/src/templates/sb_bootstrap.sh` | Zdrojovy kod bootstrapu |
| sb-ops-agent | `sb-manager/packages/sb-ops-agent/server.mjs` | Health endpoint, system info |
| Health monitor | `sb-manager/src/lib/deviceHealthMonitor.js` | VPS compliance check |
