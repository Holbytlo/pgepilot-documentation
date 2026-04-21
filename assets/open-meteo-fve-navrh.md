# Open‑Meteo pro FVE — návrh datové vrstvy, endpointů a jednoduchého algoritmu (v1)

Datum: 2026-03-30

---

## 1. Závěr v jedné minutě

Pro FVE v ČR bych to postavil takto:

1. **Živý forecast**
   - **intraday / 0–48 h:** `DWD ICON API` s `minutely_15`
   - **2–16 dní:** obecné `Forecast API` v hodinovém kroku

2. **Historie pro backfill**
   - **vyšší přesnost za poslední roky:** `Historical Forecast API`
   - **dlouhá a konzistentní historie:** `Historical Weather API` (reanalysis)

3. **Výkon počítat u sebe**
   - nebrat Open‑Meteo jako „výrobní kalkulačku“
   - brát z něj hlavně **GHI / DNI / DHI + teplotu + vítr**
   - z toho dopočítat vlastní **POA irradianci**, teplotní korekci a výkon

4. **Na začátek nepřestřelit složitost**
   - v1: fyzikální model bez ML
   - v2: bias correction podle skutečné výroby
   - v3: práce s forecast error přes `Previous Runs API`

Tohle je nejpraktičtější kompromis mezi přesností, jednoduchostí a budoucí rozšiřitelností. Forecast API umí až **16 dní**, `past_days` až **92 dní**, DWD umí v Central Europe **15min data**, Historical Forecast archivuje forecasty od **2022+** a Historical Weather dává reanalysis od **1940+**. [S1][S2][S3][S4]

---

## 2. Co Open‑Meteo pro FVE opravdu dodá

### Klíčové proměnné

Open‑Meteo má přímo proměnné, které pro FVE potřebuješ:

- `shortwave_radiation` = **GHI**
- `direct_normal_irradiance` = **DNI**
- `diffuse_radiation` = **DHI**
- `global_tilted_irradiance` = **GTI**
- `temperature_2m`
- `wind_speed_10m`
- `cloud_cover`

`shortwave_radiation` je definována jako průměrná krátkovlnná radiace za předchozí hodinu a odpovídá **globální horizontální irradianci**. `direct_normal_irradiance` je přímá irradiance na plochu kolmou ke Slunci. `diffuse_radiation` je difuzní složka. `global_tilted_irradiance` je dopočtená radiace na nakloněnou rovinu. [S1][S3]

### Důležitá nuance: GTI je užitečné, ale ne jako jediný „zdroj pravdy“

Open‑Meteo u `global_tilted_irradiance` výslovně uvádí, že výpočet používá:

- **fixní albedo 20 %**
- **isotropic sky**
- zadaný `tilt` a `azimuth`

To je dobré jako reference a sanity check, ale pro vlastní FVE model je lepší brát **GHI + DNI + DHI** a transpozici na rovinu panelu spočítat u sebe. [S1]

### Sunshine duration není hlavní fyzikální vstup

`sunshine_duration` je denní počet sekund slunečního svitu definovaný přes **DNI > 120 W/m²** podle WMO. Je to užitečné pro denní kontrolu a reporting, ale ne jako hlavní vstup pro výpočet výkonu FVE. Pro výkon je důležitější irradiance a teplota. [S1]

---

## 3. Omezení Open‑Meteo, která musí být v návrhu zohledněna

### 3.1 Je to gridové počasí, ne senzor na střeše

Open‑Meteo vybírá grid cell podle souřadnic, preferovaného režimu výběru (`cell_selection`) a výšky. Výchozí výška je vzatá z **90m DEM** a Open‑Meteo používá statistický downscaling. To znamená, že dostaneš velmi použitelné počasí pro lokalitu, ale nevidíš lokální komín, atiku, sousední strom, špínu panelů ani reálné mikro‑stínění konkrétní střechy. [S1]

### 3.2 15min data nejsou „nativní“ všude

`minutely_15` je nativně založené na modelech **NOAA HRRR** pro Severní Ameriku a **DWD ICON‑D2 / Météo‑France AROME** pro Central Europe. Mimo tyto oblasti Open‑Meteo 15min data interpoluje z hodinových. Pro ČR je to dobrá zpráva: Central Europe pokryta je. [S1][S2]

### 3.3 Historické datasety nejsou totéž

Open‑Meteo samo rozlišuje tři různé vrstvy historie:

- **Historical Weather API** = reanalysis, dlouhá a konzistentní časová řada
- **Historical Forecast API** = archiv reálných forecast modelů, vyšší lokální přesnost za poslední roky
- **Previous Runs API** = stejné forecast modely, ale s posunem lead time (co předpověď z minulého dne říkala o dnešku)

Na dlouhou klimatickou historii je lepší reanalysis. Na učení krátkodobého FV forecastu je lepší Historical Forecast. [S3][S4][S5]

### 3.4 `past_days` není náhrada za poctivý backfill

Forecast API umí `past_days` v rozsahu **0–92**, takže na pár týdnů či poslední měsíce to stačí. Na skutečný backfill ale používej `Historical Weather API` a `Historical Forecast API`. [S1]

---

## 4. Doporučené endpointy

Níže je návrh pro **jednu lokalitu FVE v ČR**.

### 4.1 Živý hodinový forecast (jednoduchý základ)

Použití:
- denní provoz
- forecast na 1–16 dní
- základní vstup pro výpočet výkonu po hodinách

Endpoint:

```http
GET https://api.open-meteo.com/v1/forecast
  ?latitude={LAT}
  &longitude={LON}
  &timezone=Europe/Prague
  &hourly=temperature_2m,wind_speed_10m,cloud_cover,shortwave_radiation,direct_normal_irradiance,diffuse_radiation,global_tilted_irradiance
  &forecast_days=16
```

Poznámky:
- `global_tilted_irradiance` bych tahal jen jako **referenční kontrolu**.
- Vlastní výpočet výkonu bych dělal z `shortwave_radiation` + `direct_normal_irradiance` + `diffuse_radiation`. [S1]

### 4.2 Intraday forecast 15 min pro ČR

Použití:
- lepší intraday odhad 0–48 h
- krátkodobé řízení baterie / přetoků / spotřeby
- jemnější průběh výkonu

Endpoint:

```http
GET https://api.open-meteo.com/v1/dwd-icon
  ?latitude={LAT}
  &longitude={LON}
  &timezone=Europe/Prague
  &minutely_15=temperature_2m,wind_speed_10m,shortwave_radiation,direct_normal_irradiance,diffuse_radiation,global_tilted_irradiance
  &forecast_minutely_15=192
  &past_minutely_15=96
```

Důvod:
- DWD endpoint pro Central Europe přímo používá **ICON‑D2** a umí **15min solární proměnné**. [S2]

Praktická interpretace:
- `forecast_minutely_15=192` = 48 h dopředu
- `past_minutely_15=96` = posledních 24 h zpět

### 4.3 Historická „skutečnost“ počasí / reanalysis

Použití:
- dlouhý backfill
- porovnání rok proti roku
- robustní baseline, když teprve začínáš ukládat data

Endpoint:

```http
GET https://archive-api.open-meteo.com/v1/archive
  ?latitude={LAT}
  &longitude={LON}
  &timezone=Europe/Prague
  &start_date={YYYY-MM-DD}
  &end_date={YYYY-MM-DD}
  &hourly=temperature_2m,wind_speed_10m,cloud_cover,shortwave_radiation,direct_normal_irradiance,diffuse_radiation
```

Doporučení:
- pro **dlouhou konzistentní historii** používej v Historical Weather API dataset typu **ERA5** nebo **ERA5‑Seamless**
- pokud chceš čistě dlouhou klimatickou konzistenci, Open‑Meteo přímo doporučuje **ERA5 / ERA5‑Land**; pro FVE je ale důležité, že **ERA5‑Land samo o sobě není vhodné jako jediný zdroj solar radiation**, zatímco **ERA5** a **ERA5‑Seamless** solar radiation mají. [S3]

### 4.4 Historické forecasty pro lepší backfill posledních let

Použití:
- trénink a validace modelu na realistických forecast datech
- lepší lokální přesnost než čistá reanalysis v posledních letech
- konzistence s tím, co budeš používat v produkci

Endpoint:

```http
GET https://historical-forecast-api.open-meteo.com/v1/forecast
  ?latitude={LAT}
  &longitude={LON}
  &timezone=Europe/Prague
  &start_date={YYYY-MM-DD}
  &end_date={YYYY-MM-DD}
  &hourly=temperature_2m,wind_speed_10m,cloud_cover,shortwave_radiation,direct_normal_irradiance,diffuse_radiation,global_tilted_irradiance
```

Open‑Meteo u tohoto API výslovně píše, že je vhodné pro trénování ML a kombinaci forecast dat do optimalizovaných predikcí. Data jsou dostupná podle modelu od **2021/2022**. [S4]

### 4.5 Previous Runs API (volitelné, až v další fázi)

Použití:
- otázka „co včerejší forecast tvrdil o dnešku?“
- forecast error / bias correction
- zlepšení D+1 a D+2

Endpoint host:

```http
GET https://previous-runs-api.open-meteo.com/v1/forecast
```

Open‑Meteo generuje proměnné ve stylu:

- `temperature_2m`
- `temperature_2m_previous_day1`
- `temperature_2m_previous_day2`
- ...

Oficiální ukázková URL takto vrací `temperature_2m_previous_day1..5`; totéž schéma je určené i pro další proměnné. Dokumentace zároveň ukazuje, že Previous Runs API podporuje i **solar radiation variables** včetně `shortwave_radiation`, `diffuse_radiation`, `direct_normal_irradiance` a `global_tilted_irradiance`. [S5][S11]

Poznámka:
- tuhle vrstvu bych **nedával do MVP**, ale nechal ji pro fázi kalibrace forecastu.

### 4.6 Satellite Radiation API (volitelné, V2)

Použití:
- near‑real irradiance reference
- kontrola, zda forecast nerelaxuje cloud field příliš optimisticky/pesimisticky
- lepší „co se děje teď“ pohled na radiaci

Open‑Meteo má samostatné Satellite Radiation API s GHI/DNI/DHI/GTI. V Evropě je v archivu například **SARAH3 od 1983** a novější EUMETSAT / MTG vrstvy s kratším zpožděním. Není to nutné do první verze, ale pro V2 je to velmi zajímavé. [S8]

---

## 5. Co bych ukládal do databáze

## 5.1 `meteo_forecast_hourly`

Jedna řádka = jeden forecast snapshot pro jeden cílový čas.

Doporučené sloupce:

- `site_id`
- `fetched_at_utc`
- `target_time_local`
- `source_endpoint` (`forecast`, `dwd-icon`, `historical-forecast`, `previous-runs`)
- `model_hint` (pokud budeš model pinovat)
- `timezone`
- `latitude`
- `longitude`
- `elevation`
- `temperature_2m_c`
- `wind_speed_10m_ms`
- `cloud_cover_pct`
- `ghi_wm2` (`shortwave_radiation`)
- `dni_wm2` (`direct_normal_irradiance`)
- `dhi_wm2` (`diffuse_radiation`)
- `gti_ref_wm2` (`global_tilted_irradiance`, volitelné)
- `raw_json` (volitelně, pro audit)

Indexy:
- `(site_id, fetched_at_utc, target_time_local)`
- `(site_id, target_time_local)`

## 5.2 `meteo_actual_hourly`

Jedna řádka = „skutečnost“ z Historical Weather nebo Satellite Radiation.

Sloupce podobné forecast tabulce, ale místo `fetched_at_utc` jen:

- `valid_time_local`
- `dataset` (`archive-era5`, `archive-era5-seamless`, `satellite`, ...)

## 5.3 `pv_actual`

- `site_id`
- `valid_time_local`
- `power_w`
- `energy_wh`
- `inverter_limit_w` (volitelně)
- `curtailment_flag` (volitelně)
- `battery_charge_w` / `battery_discharge_w` (volitelně)
- `grid_export_w` / `grid_import_w` (volitelně)

Pro seriózní kalibraci je ideální mít **5–15min výrobu**, ne jen denní sumy.

---

## 6. Doporučená implementační strategie

### MVP

1. každou hodinu tahat **Forecast API hourly**
2. každých 15 minut tahat **DWD ICON minutely_15**
3. 1× denně dotahovat **včerejší actual** z `archive-api`
4. jednorázově udělat backfill posledních 2–4 let
5. uložit vlastní skutečnou výrobu

### Proč takto

- forecast a actual budou oddělené
- budeš mít data pro ex‑post analýzu přesnosti
- nebudeš trénovat model na datech „po bitvě“, když ve skutečnosti chceš predikovat z forecastu

### Backfill: po jakých dávkách stahovat

Open‑Meteo uvádí, že požadavky s více než **10 proměnnými** nebo přes období delší než **2 týdny** mohou být účtovány jako více API callů. Proto je rozumné stahovat backfill po **měsících** nebo po **14 dnech**, podle toho, co je pohodlnější pro parser a retry logiku. [S6]

---

## 7. Návrh algoritmu pro výpočet výkonu

Níže jsou dvě verze.

## 7.1 Varianta A — nejjednodušší a rychlá

Použij rovnou `global_tilted_irradiance` jako radiaci na rovinu panelu.

### Vstupy

- `P_stc_total = panel_count * panel_wp`
- `beta = sklon panelu`
- `gamma_p = azimut panelu`
- `gamma_pmp` = teplotní koeficient výkonu z datasheetu (typicky záporný)
- `NOCT` nebo `NMOT` z datasheetu
- `eta_balance` = systémová účinnost DC→AC po ztrátách
- `P_ac_limit` = limit střídače / exportu

### Výpočet

1. `G_poa = global_tilted_irradiance`
2. odhad teploty článku, např.

```text
T_cell ≈ T_air + (NOCT - 20) / 800 * G_poa
```

3. DC výkon:

```text
P_dc = P_stc_total * (G_poa / 1000) * (1 + gamma_pmp * (T_cell - 25))
```

4. AC výkon:

```text
P_ac = min(P_dc * eta_balance, P_ac_limit)
```

5. denní energie = numerická integrace po čase

```text
E_day_Wh = Σ P_ac(t) * Δt_hours
```

### Výhody

- velmi jednoduché
- rychlá implementace
- dobré jako první baseline

### Nevýhody

- závisí na GTI od Open‑Meteo, které používá fixní albedo 20 % a isotropic sky
- hůř se kontroluje, co přesně udělal transpoziční model

## 7.2 Varianta B — doporučená jednoduchá fyzika

Tahle varianta je za mě správná v1 pro „výkon počítám u sebe“.

### Meteorologické vstupy

- `GHI = shortwave_radiation`
- `DNI = direct_normal_irradiance`
- `DHI = diffuse_radiation`
- `T_air = temperature_2m`
- `V_wind = wind_speed_10m`

### Krok 1: sluneční geometrie

Pro každý timestamp spočítat:

- sluneční zenit `theta_z`
- sluneční azimut `gamma_s`

### Krok 2: úhel dopadu na panel

Při sklonu panelu `beta` a panelovém azimutu `gamma_p`:

```text
cos(theta_i) = cos(theta_z)*cos(beta) + sin(theta_z)*sin(beta)*cos(gamma_s - gamma_p)
```

Použij:

```text
cos_i = max(0, cos(theta_i))
```

### Krok 3: irradiance na rovinu panelu (POA)

Jednoduchá isotropní aproximace:

```text
POA_beam    = DNI * cos_i
POA_diffuse = DHI * (1 + cos(beta)) / 2
POA_ground  = GHI * rho_ground * (1 - cos(beta)) / 2
POA_total   = POA_beam + POA_diffuse + POA_ground
```

Doporučení pro v1:

```text
rho_ground = 0.2
```

Pak si můžeš porovnat `POA_total` s Open‑Meteo `global_tilted_irradiance` jako kontrolu.

### Krok 4: teplota článku

Jednoduchý v1 odhad:

```text
T_cell ≈ T_air + (NOCT - 20) / 800 * POA_total
```

Lepší v2:
- doplnit vliv větru (`wind_speed_10m`)
- případně přejít na detailnější teplotní model

### Krok 5: DC výkon

```text
P_dc = P_stc_total * (POA_total / 1000) * (1 + gamma_pmp * (T_cell - 25))
```

### Krok 6: systémové ztráty a AC limit

```text
P_ac_preclip = P_dc * eta_balance
P_ac = min(P_ac_preclip, P_ac_limit)
```

Kde `eta_balance` může zahrnovat například:

- kabely
- mismatch
- měnič
- DC/DC převody
- lehkou rezervu na znečištění

Na úplný začátek je praktické dát např. jednu agregovanou účinnost a později ji rozdělit.

---

## 8. Co bych dělal s historií, když začínáš právě teď

### Cíl A — chci dlouhou historii počasí

Použij `archive-api` a stáhni minimálně:

- poslední 2–4 roky
- hodinově:
  - `temperature_2m`
  - `wind_speed_10m`
  - `cloud_cover`
  - `shortwave_radiation`
  - `direct_normal_irradiance`
  - `diffuse_radiation`

To ti dá okamžitou historii pro analýzy a simulaci výroby.

### Cíl B — chci historii forecastů, ne jen „skutečné počasí“

Použij `historical-forecast-api` a stáhni poslední 2–4 roky. To je správná volba, pokud časem chceš hodnotit, jak by fungoval skutečný forecast výroby dopředu. Open‑Meteo přímo uvádí, že Historical Forecast je vhodnější pro vyšší přesnost posledních let a že právě kombinace Historical Forecast + Previous Runs je dobrá pro optimalizaci forecastů pomocí ML. [S4]

### Cíl C — chci od teď budovat vlastní databázi

Od dneška ukládej:

- živý forecast každou hodinu
- 15min intraday forecast každých 15 minut
- actual weather 1× denně zpětně
- vlastní výrobu průběžně

Za 2–3 měsíce budeš mít použitelný dataset na kalibraci.

---

## 9. Co bych do první verze nedělal

- žádné složité ML nad počasím hned na začátku
- žádné používání `sunshine_duration` jako hlavního vstupu
- žádné spoléhání na jeden jediný koeficient bez teplotní korekce
- žádné míchání forecastu a historical actual do jednoho datasetu bez označení zdroje
- žádné „best match“ bez uložení informace, jaký endpoint a jaká vrstva byla použita

---

## 10. Licence, limity a plány

### Free API

Open‑Meteo pro free non‑commercial použití uvádí limity:

- **10 000 callů / den**
- **5 000 callů / hodinu**
- **600 callů / minutu**

Free API je jen pro **non‑commercial use** a bez service guarantees. [S6][S7]

### Placené API

Oficiální pricing stránka potvrzuje:

- commercial use licence
- dedicated customer API servery
- vyšší spolehlivost
- Standard: **1M callů / měsíc**
- Professional: **5M callů / měsíc**
- Enterprise: **>50M callů / měsíc**
- customer API používá jiné domény, např. `customer-api.open-meteo.com`, a `apikey`

Pricing stránka také výslovně říká, že **historical, climate a ensemble data jsou pro komerční použití až od Professional plánu**. Tabulka zároveň ukazuje, že **Previous Model Runs API není ve Standard plánu**, ale je v Professional a Enterprise. [S6]

### Cena v EUR/USD

V parseovatelném textu aktuální oficiální pricing stránky se mi podařilo ověřit **feature gating a měsíční call capacity**, ale ne samotné číselné ceny plánů. Proto je potřeba aktuální cenu ověřit přímo na oficiální pricing stránce / customer portalu. [S6]

---

## 11. Doporučené pořadí implementace

### Fáze 1 — běžící základ

- Forecast API hourly
- DWD minutely_15
- vlastní výpočet výkonu z GHI/DNI/DHI
- ukládání do DB

### Fáze 2 — kalibrace

- porovnání s reálnou výrobou
- doladění `eta_balance`
- doladění teplotního modelu
- clipping / export limit / baterie

### Fáze 3 — forecast quality

- Historical Forecast API
- Previous Runs API
- bias correction pro D+1 a D+2

### Fáze 4 — volitelně lepší radiace

- Satellite Radiation API jako reference/nowcast vrstva

---

## 12. Zadání pro programovací chat

Níže je hotové zadání, které můžeš zkopírovat do jiného chatu.

```text
Navrhni a implementuj službu pro FVE forecast, která používá Open-Meteo jen jako zdroj meteorologických vstupů a výkon počítá interně.

Cíl:
- vstup: GPS, sklon, azimut, počet panelů, výkon panelu Wp, teplotní koeficient gamma_pmp, NOCT/NMOT, účinnost/ztráty systému, limit střídače
- výstup: forecast výkonu po 15 min / hodině a denní energie

Použij tyto endpointy:
1) Live hourly forecast:
   GET https://api.open-meteo.com/v1/forecast
   hourly=temperature_2m,wind_speed_10m,cloud_cover,shortwave_radiation,direct_normal_irradiance,diffuse_radiation,global_tilted_irradiance
   timezone=Europe/Prague
   forecast_days=16

2) Live intraday 15 min for Czech Republic:
   GET https://api.open-meteo.com/v1/dwd-icon
   minutely_15=temperature_2m,wind_speed_10m,shortwave_radiation,direct_normal_irradiance,diffuse_radiation,global_tilted_irradiance
   timezone=Europe/Prague
   forecast_minutely_15=192
   past_minutely_15=96

3) Historical actual weather:
   GET https://archive-api.open-meteo.com/v1/archive
   hourly=temperature_2m,wind_speed_10m,cloud_cover,shortwave_radiation,direct_normal_irradiance,diffuse_radiation
   timezone=Europe/Prague
   start_date / end_date

4) Historical forecasts:
   GET https://historical-forecast-api.open-meteo.com/v1/forecast
   hourly=temperature_2m,wind_speed_10m,cloud_cover,shortwave_radiation,direct_normal_irradiance,diffuse_radiation,global_tilted_irradiance
   timezone=Europe/Prague
   start_date / end_date

Databáze:
- meteo_forecast_hourly(site_id, fetched_at_utc, target_time_local, source_endpoint, model_hint, latitude, longitude, elevation, temperature_2m_c, wind_speed_10m_ms, cloud_cover_pct, ghi_wm2, dni_wm2, dhi_wm2, gti_ref_wm2, raw_json)
- meteo_actual_hourly(site_id, valid_time_local, dataset, latitude, longitude, elevation, temperature_2m_c, wind_speed_10m_ms, cloud_cover_pct, ghi_wm2, dni_wm2, dhi_wm2, raw_json)
- pv_actual(site_id, valid_time_local, power_w, energy_wh, inverter_limit_w, curtailment_flag)

Algoritmus v1:
1. Spočítej sluneční pozici z GPS + času.
2. Vezmi GHI=shortwave_radiation, DNI=direct_normal_irradiance, DHI=diffuse_radiation.
3. Spočítej úhel dopadu na panel.
4. Spočítej POA_total = POA_beam + POA_diffuse + POA_ground.
5. Odhadni T_cell z T_air a POA_total přes NOCT/NMOT aproximaci.
6. Spočítej P_dc = P_stc_total * (POA_total/1000) * (1 + gamma_pmp * (T_cell - 25)).
7. Aplikuj ztráty a limit střídače.
8. Integruj výkon na energii.
9. Ulož hourly/minutely forecast i daily sums.

Důležité:
- global_tilted_irradiance používej jen jako kontrolu, ne jako jediný vstup
- všechny časy vracej a ukládej v timezone Europe/Prague
- odděl forecast data od actual dat
- připrav backfill historie po měsíčních dávkách
- připrav retry, timeouty a idempotentní ukládání
```

---

## 13. Zdroje (oficiální)

- [S1] Weather Forecast API — https://open-meteo.com/en/docs
- [S2] DWD ICON API — https://open-meteo.com/en/docs/dwd-api
- [S3] Historical Weather API — https://open-meteo.com/en/docs/historical-weather-api
- [S4] Historical Forecast API — https://open-meteo.com/en/docs/historical-forecast-api
- [S5] Previous Runs API — https://open-meteo.com/en/docs/previous-runs-api
- [S6] Pricing — https://open-meteo.com/en/pricing
- [S7] Terms — https://open-meteo.com/en/terms
- [S8] Satellite Radiation API — https://open-meteo.com/en/docs/satellite-radiation-api
- [S9] Model Updates — https://open-meteo.com/en/docs/model-updates
- [S10] Licence — https://open-meteo.com/en/licence
- [S11] Oficiální vygenerovaná ukázková URL z DWD/Previous Runs docs (přes „Open in new tab“ v dokumentaci); potvrzuje hosty a příklad syntaxe parametrů.

