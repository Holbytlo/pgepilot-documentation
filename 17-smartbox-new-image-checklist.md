# 17 -- SmartBox New Image Checklist

> Operativni checklist pro vytvoreni a oziveni noveho SmartBox image.
> Toto je git-backed kanonicky vstup pro nove SD image a nahrazuje session-only poznamky a ad-hoc MP135 runbooky jako hlavni zdroj pravdy.
> Posledni aktualizace: 2026-04-14

---

## Kdy pouzit tento dokument

Pouzij ho pri kazdem novem boxu:

- kdyz vytvaris novy `MP135` nebo `RPi` image
- kdyz flashujes novou SD kartu
- kdyz delas prvni boot a provisioning
- kdyz pripravujes box pred odjezdem k zakaznikovi

Tento dokument je prakticky checklist.
Detailni vysvetleni zustava v:

- `02-smartbox-sbc.md` — architektura a runtime pravidla
- `14-smartbox-provisioning.md` — kompletni provisioning flow
- `16-provisioning-issues.md` — backlog a opakujici se pasti

---

## 1. Vstupy pred buildem

Pred vytvorenim image musi byt jasne:

- platforma: `MP135` nebo `RPi`
- `sb_id`, `deviceLabel`, `n`
- cilovy `alias` a lidsky `notes`
- cilovy `collection_point_id`, `device_id`, `smartbox_id`, `machine_id`
- typ primarniho konektoru: `GoodWe`, `Deye`, nebo jiny
- jestli bude box commissioning delat pres `Ethernet`, `Wi-Fi`, nebo docasne `USB tethering`
- typ fyzickeho displeje

Pravidlo:

- `alias` patri do alias pole
- `notes` jsou jen lidsky popis lokality
- box-side naming, cloud naming a display naming se nesmi rozjet

---

## 2. Registrace v sb-manageru

Pred flashi musi byt box zalozeny v `sb-manageru`:

1. vytvorit zarizeni se spravnym `n`
2. vyplnit `alias` a `notes`
3. overit port plan podle `n`
4. vygenerovat provisioning token
5. vygenerovat bootstrap script

Pokud jde o customer box, musi byt uz predem jasne:

- control domain
- collection point
- canonical machine UUID pro inverter

---

## 3. Build image

Pouzivej buildery z `sb-manager/tools/sb/`.

Pravidla:

- velke image a temp artefakty patri na externi disk, ne na hlavni disk Macu
- nove `GoodWe` image maji defaultne generovat `dhcp` auto-scan konfiguraci, ne placeholder statickou IP
- display runtime se ma brat z kanonicke live/repo vetve, ne z ad-hoc nouzove varianty

Pro nove image zkontroluj:

- spravny `deviceLabel`
- spravny `alias`
- spravny `notes`
- spravny typ platformy
- spravny defaultni primarni konektor

---

## 4. Flash SD karty

Po flashi over:

- image se zapsal bez verify erroru
- karta se korektne odpojila
- box bootuje z nove SD
- serial konzole je dostupna

Preferovany postup:

1. vytvorit image
2. flashnout SD
3. prvni boot udelat na stole se serialem
4. az pak resit commissioning site-specific konektoru

---

## 5. Prvni boot a sit

Preferovane poradi:

1. `Ethernet`
2. `USB tethering`
3. `Wi-Fi`

### Wi-Fi kompatibilita na MP135

Aktualni kernel na `MP135` (`5.15.118`) ma problem s nekterymi starsimi Ralink dongly.

Konretne:

- `RT5370` (`148f:5370`) neni na current image pouzitelny
- `rt2800usb` modul je pritomny, ale kernel ma vypnute `CONFIG_RT2800USB_RT53XX`
- vysledek: dongle je videt v `lsusb`, ale nevznikne `wlan0` a `nmcli` hlasi `WIFI-HW missing`

Z toho plyne:

- nepocitat `RT5370` jako standardni commissioning Wi-Fi dongle
- pro nouzi je lepsi `Ethernet`, `USB tethering`, nebo jiny kompatibilni dongle
- pokud ma byt `RT5370` podporovany standardne, je potreba novy kernel build

---

## 6. Bootstrap a tunel

Po siti musi probehnout:

1. bootstrap script ze `sb-manageru`
2. registrace SSH tunnel key
3. start `sb-ops-agent`
4. start `ra-tunnel.service`

Hned over:

- `sb-manager` vidi box jako `online`
- funguje `ops`
- funguje `web`
- funguje `api`
- funguje `health`

Pokud tunel jen otevre porty, ale HTTP/SSH na nich visi bez odpovedi, nepovazovat to za hotove. To je degradovany stav, ne uspesny provisioning.

---

## 7. Nasazeni aplikacniho stacku

Po bootstrapu musi byt dorovnane:

- `sb` runtime
- `.venv`
- systemd unity
- ownership
- display service

Minimalni kontroly:

- `python3 -m venv` funguje
- vsechny potrebne Python deps jsou nainstalovane
- `energity-*` services nespadaji na import chyby
- po manualnim copy probehne `chown` normalizace

---

## 8. Primarni konektor

Kanonicky commissioning model:

- `sb-manager` nastavuje primarni konektor boxu
- zapis jde do box-side `devices_config.yaml` pres SB `config-api`
- po ulozeni se restartuje `energity-device-controller`

### GoodWe

Default pro novy box:

- `mode: dhcp`
- bez `mac_address`
- `unit_id: 247`
- `discovery_cache_ttl: 300`

Pravidla:

- kdyz je v siti presne jeden GoodWe, je dovolena auto-detekce
- kdyz je GoodWe vic, je to zamerne `ambiguous`
- v ambiguous stavu se musi prejit na:
  - `static host`
  - nebo `dhcp + mac_address`

---

## 9. Co musi byt hotove pred odjezdem k zakaznikovi

Pred predanim boxu musi byt overene:

- box ma spravne jmeno v `sb-manageru`
- box-side naming odpovida `sb-manageru`
- display vypada kanonicky pro danou platformu
- rootfs je expandovany a ma dost mista
- `LocalDB` je spravne hlidana v health/dashboard logice
- public routy `web/ops/api/health` odpovidaji
- placeholder connector config je bud vedome ponechana pro commissioning, nebo uz nahrazena realnou

Pokud box jeste nema customer data, musi to byt explicitne oznacene jako:

- runtime standardized
- commissioning pending

---

## 10. Opakujici se pasti po kazdem novem image

Vzdy znovu zkontroluj:

- watchdog
- `PubkeyAuthentication`
- `python3-venv`
- `nodejs`
- ownership po manualnim deployi
- systemd path correctness
- display runtime drift
- health route drift
- Wi-Fi dongle kompatibilitu

Tohle nejsou jednorazove bugy. Jsou to veci, ktere se maji overit pri kazdem novem image.

---

## 11. Source of truth

Pro novy image plati tato hierarchie:

1. `17-smartbox-new-image-checklist.md` — operativni checklist
2. `14-smartbox-provisioning.md` — detailni provisioning flow
3. `16-provisioning-issues.md` — backlog a otevrene pasti
4. `02-smartbox-sbc.md` — architektura a runtime standard

Session-only soubory a lokalni handoffy maji byt pomocne, ne hlavni zdroj pravdy.
