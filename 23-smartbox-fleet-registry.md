# 23 -- SmartBox Fleet Registry

> Canonical operational registry for SmartBox labels, aliases, customer/site names, expected inverter type, and current provisioning state.
>
> Last updated: 2026-04-28

## Scope

This document records the agreed naming for the active SmartBox fleet. It is not a secrets store and must not contain SSH passwords, tokens, private keys, or customer credentials.

For live operational state, verify against:

- `sb-manager` production DB on `ra-energity.cz`
- physical device hostname and `/opt/energity/sb-ops-agent/device_alias.txt`
- reverse tunnel ports on `ra-energity.cz`
- PgePilot cloud SmartBox identity bundle

## Naming Rules

| Field | Rule |
|---|---|
| `device_label` | Stable technical label: `sb<N>` |
| `n` | Numeric port block ID, equal to `<N>` |
| `alias` | Short lowercase DNS/UI slug, no spaces or accents |
| `notes` | Human-readable site name only |
| customer/domain | Portfolio/customer name in PgePilot |
| collection point | Physical location/site in PgePilot |

Port block formula:

```text
base = 20000 + (n - 1) * 10
+0 = UI/nginx, +2 = SSH, +3 = config-api, +4 = rpc-api, +5 = ops-agent, +9 = health
```

## Active Registry

| SB | n | Alias | Notes / Site | Customer / Domain | Inverter target | Hardware class | Current state |
|---|---:|---|---|---|---|---|---|
| sb7 | 7 | `chlumzs` | Chlumcany ZS | Chlumcany | Deye | MP135 | Customer site; remote availability depends on customer network. |
| sb9 | 9 | `borik4` | Borik Chata 4 | Borik Severni | GoodWe planned | RPi | Prepared for Borik deployment. |
| sb11 | 11 | `kosticekd` | Kostice KD | Kostice OU | 2x GoodWe 25 kW, master/slave | RPi | Active and used as RPi display/runtime standard. |
| sb13 | 13 | `listanykd` | Listany KD | Listany | 2x SoLaX | RPi | Active reference for SoLaX class. |
| sb14 | 14 | `kosticeou` | Kostice OU | Kostice OU | 1x GoodWe | RPi | Prepared for Kostice OU. |
| sb15 | 15 | `zbrasinou` | Zbrasin OU | Zbrasin | 2x SoLaX | RPi | Active on LAN/VPS; SoLaX devices detected and configured. |
| sb16 | 16 | `citolibyzs` | Citoliby ZS | Citoliby | 2x SoLaX | RPi | Active on LAN/VPS; runtime, cloud identity, tunnel, and SoLaX placeholders configured. |
| sb17 | 17 | `citolibyhs` | Citoliby HS | Citoliby | 2x SoLaX | RPi | Active on LAN/VPS; runtime, cloud identity, tunnel, and SoLaX placeholders configured. |

## Devices Outside This Wave

| SB | Status |
|---|---|
| sb1 | Explicitly excluded from the 2026-04-27 standardization wave. |
| sb4 | Dev/home reference box; intentionally ignored for the current customer deployment wave. |
| sb8 | Development/reserve (`MICHAL vyvoj`). |
| sb10 | Reserve/unassigned in the current wave. |

## 2026-04-28 Verification Notes

`sb15`:

- `sb-manager` row: `n=15`, label `sb15`, alias `zbrasinou`, notes `Zbrasin OU`, status `ACTIVE`
- reverse tunnel active on base `20140`
- SoLaX Modbus TCP devices found at site LAN and configured as `inverter` + `inverter2`
- driver compatibility fix for current `pymodbus` was deployed on-box and committed to `Holbytlo/sb` branch `codex/local-sb-wip-20260427`

`sb16`:

- `sb-manager` row: `n=16`, label `sb16`, alias `citolibyzs`, notes `Citoliby ZS`, status `ACTIVE`
- observed on staging LAN as `pge-sb16` at `192.168.0.125`, MAC `88:a2:9e:54:6c:8a`
- reverse tunnel active on base `20150`
- VPS SSH active on `20152`
- nginx `/health` active on `20150`
- `sb-ops-agent` health active on `20155`, `allUp=true`, 5/5 internal services
- `devices_config.yaml` contains 2x disabled SoLaX Modbus TCP placeholders (`inverter`, `inverter2`) with Citoliby ZS machine IDs

`sb17`:

- `sb-manager` row: `n=17`, label `sb17`, alias `citolibyhs`, notes `Citoliby HS`, status `ACTIVE`
- observed on staging LAN as `pge-sb17` at `192.168.0.129`, MAC `88:a2:9e:54:6c:ce`
- reverse tunnel active on base `20160`
- VPS SSH active on `20162`
- nginx `/health` active on `20160`
- `sb-ops-agent` health active on `20165`, `allUp=true`, 5/5 internal services
- `devices_config.yaml` contains 2x disabled SoLaX Modbus TCP placeholders (`inverter`, `inverter2`) with Citoliby HS machine IDs

## Operational Requirement For New Images

Before a SmartBox leaves the bench, verify all of these from the Mac and from `ra-energity.cz`:

- hostname equals `pge-sb<N>` or the agreed device hostname
- a known SSH key works for root or the documented operator account
- `ra-tunnel` is enabled and active
- the expected VPS port block is listening
- `sb-ops-agent` answers on `<base+5>/health`
- `sb-manager` shows the correct label, alias, notes, and online state
- physical display uses autodetection and matches the shared RPi/MP135 dashboard standard

If only SSH port is open and no known credential works, the device is not field-ready even if it appears in DHCP or Bonjour.
