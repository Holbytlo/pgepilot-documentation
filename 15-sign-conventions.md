# 15 -- Konvence znamenek pro vykon (Sign Conventions)

> Jednotna definice co znamena kladne a zaporne cislo u kazdeho typu mereni.
> Pro programatory: VSECHNY drivery MUSI normalizovat na tuto konvenci PRED tim nez data posle dal.
> Posledni aktualizace: 2026-04-12

---

## Standardni konvence (cilovy stav)

Vsechny hodnoty v systemu energity (sensor_data.db, API, dashboard) MUSI pouzivat tuto konvenci:

| Velicina | Kladne (+) | Zaporne (-) | Jednotka |
|----------|-----------|-------------|----------|
| **battery_power** | Nabijeni (energie do baterie) | Vybijeni (energie z baterie) | W |
| **grid_power** | Nakup ze site (import) | Prodej do site (export) | W |
| **pv_power** | Vykon FVE (vzdy kladne) | N/A (vzdy >= 0) | W |
| **load_power** | Spotreba domu (vzdy kladne) | N/A (vzdy >= 0) | W |
| **inverter_power** | Celkovy vykon stridace | N/A | W |

### Mnemotechnika

- **Baterie**: `+` = do baterie (nabijim), `-` = z baterie (vybijim)
- **Sit**: `+` = ze site (kupuju), `-` = do site (prodavam)
- Pohled z domu: `+` = "prichazi ke mne", `-` = "odchazi ode me"

---

## Realna konvence na stridacich (jak to driver cte)

### GoodWe (ET/EH/BT series)

| Registr | Adresa | GoodWe konvence | Nase konvence | Potreba invertovat? |
|---------|--------|----------------|---------------|---------------------|
| Battery1 Power | 35182 (S32) | **+ = vybijeni, - = nabijeni** | + = nabijeni | **ANO** (`* -1`) |
| Battery2 Power | 35264 (S32) | **+ = vybijeni, - = nabijeni** | + = nabijeni | **ANO** (`* -1`) |
| Meter Total Active Power | 36008 (S32) | **+ = export, - = import** (nektere FW) nebo **uint16 rollover** | + = import | **OVERIT per FW verze** |
| Total Load Power | 35172 (S32) | + = spotreba | + = spotreba | NE |
| PV1/PV2 Power | 35105/35109 (U32) | + = vykon | + = vykon | NE |
| Total Inverter Power | 35138 (S32) | + = vykon | + = vykon | NE |

**DULEZITE:** GoodWe meter_total_active_power (registr 36008) na nekterych firmware verzich vraci uint16 s rolloverem (hodnoty > 32767 jsou zaporne). Je potreba `if val > 32767: val = val - 65536`.

#### Overeno na realnem HW (2026-04-12):
```
battery1_power = -1750 W  →  SOC stoupá (49→50%)  →  záporné = nabíjení u GoodWe
battery1_power = +571 W   →  SOC klesá            →  kladné = vybíjení u GoodWe
```

### Deye (SUN-xK-SG04LP3-EU)

| Registr | Deye konvence | Nase konvence | Potreba invertovat? |
|---------|---------------|---------------|---------------------|
| Battery Power | **+ = nabijeni, - = vybijeni** | + = nabijeni | **NE** (uz sedi) |
| Grid Power | **+ = import, - = export** | + = import | **NE** (uz sedi) |
| PV Power | + = vykon | + = vykon | NE |
| Load Power | + = spotreba | + = spotreba | NE |

**Deye pouziva nasi konvenci nativne** — zadna inverze neni potreba.

### SoLaX

| Registr | SoLaX konvence | Potreba invertovat? |
|---------|---------------|---------------------|
| Battery Power | + = vybijeni, - = nabijeni (jako GoodWe) | **ANO** |
| Grid Power | + = export, - = import | **ANO** |

### Victron

| Registr | Victron konvence | Potreba invertovat? |
|---------|-----------------|---------------------|
| Battery Power | + = vybijeni, - = nabijeni | **ANO** |
| Grid Power | + = import, - = export | NE |

---

## Kde se normalizace provadi

### Aktualni stav (2026-04-12): NENORMALIZUJE SE

Drivery (`goodwe_inverter.py`, `deye_inverter.py` atd.) ctou raw registery a posilaji je dal BEZ normalizace. Kazdy konzument (dashboard, web UI, cloud) musi sam vedet jakou konvenci ma dany stridac.

### Cilovy stav: normalizace v driveru

Kazdy driver (v `read_data()` nebo v poll function executoru) MUSI invertovat hodnoty tak, aby vystup odpovidel nasi standardni konvenci (viz tabulka nahore).

**Implementace:**

```python
# V goodwe_inverter.py:
def normalize_battery_power(self, raw_value):
    # GoodWe: positive = discharging, negative = charging
    # Our convention: positive = charging
    return -raw_value

def normalize_grid_power(self, raw_value):
    # GoodWe meter: may need uint16->int16 conversion + sign flip
    if raw_value > 32767:
        raw_value = raw_value - 65536
    # GoodWe: positive = export, Our: positive = import
    return -raw_value
```

```python
# V deye_inverter.py:
# Deye uz pouziva nasi konvenci — zadna zmena
def normalize_battery_power(self, raw_value):
    return raw_value  # uz sedi

def normalize_grid_power(self, raw_value):
    return raw_value  # uz sedi
```

### Kde pridat normalizaci v kodu

1. `device_controller/drivers/devices/goodwe_inverter.py` — pridat `normalize_battery_power()`, `normalize_grid_power()` a zavolat po cteni registru
2. `device_controller/drivers/devices/solax_inverter.py` — stejne
3. `device_controller/drivers/devices/deye_inverter.py` — pass-through (uz sedi)
4. `device_controller/drivers/devices/victron_inverter.py` — battery: invertovat, grid: OK

### Task

| ID | Task | Priorita | Effort |
|----|------|----------|--------|
| H16 | **Normalizace znamenek v driverech** | HIGH | Small (1-2h) |

Bez tohoto kazdy novy driver, dashboard, nebo cloud integrace musi rucne resit konvenci. S normalizaci v driveru staci jednou a vsechno dal funguje automaticky.

---

## Pouziti v display dashboardu (docasne reseni)

Dokud normalizace neni v driveru, dashboard (`display_dashboard.py`) resi konvenci sam:

```python
# GoodWe: negative = charging → flip for display
if batt < -50:
    batt_label = "NABIJENI"   # charging
    batt_sign = "+"
elif batt > 50:
    batt_label = "VYBIJENI"   # discharging
    batt_sign = "-"
```

**Az bude normalizace v driveru, dashboard logika se zjednodusi na:**
```python
if batt > 50:
    batt_label = "NABIJENI"
    batt_sign = "+"
elif batt < -50:
    batt_label = "VYBIJENI"
    batt_sign = "-"
```
