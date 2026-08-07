# EPS_V1

Electrical Power System for PVDX — battery charging, regulation, and power distribution.

## Status
- Design phase: layout
- Current rev: rev-1.2
- Contributors: Kelly Lin, Jayla Hsiung, Dean Zweiman, Nick Cavallo

## System overview

Four 2S battery packs (8x Tenergy 30005-0 18650 cells) feed a charge/regulation chain built entirely from dedicated power-management ICs, with no onboard MCU — charging, cell balancing, and telemetry are handled in hardware and reported to the compute board over I2C.

**Board sheets:**

- **Battery Pack 1–4**: one **MP2672AGD-0000-Z** charger per pack handles charging, discharging, and cell balancing without needing a microcontroller to manage the process. One **MAX17205** fuel gauge per pack reports state-of-charge and battery telemetry to the compute board over its own I2C pair (SDA/SCL), so each of the 4 packs has an independent data channel — a single fuel-gauge/charger failure doesn't take down monitoring for the other packs.
- **Charging Inputs**: the **MP28167GQ-Z** converts solar input down to a fixed 5V for charging. A **TPS2121RUXR** power mux sits between the solar and external-5V inputs and picks whichever is present, prioritizing external input — this is what lets the board charge normally on the bench (external 5V) without needing solar illumination, while still working correctly on orbit.
- **Converters**: three fixed-output regulators step the ~6–8.4V battery bus down to the rails other subsystems actually need: **TPS62133RGTR** (5V), **TPS62132RGTR** (3.3V), and **TPS61088RHLR** (12V boost, needed since 12V is above the battery bus voltage).
- **Connectors**: battery pack I/O, the RBF (remove-before-flight) pin, inhibit switches (mechanically disconnect batteries from load/ground when pressed — a safety requirement, not just a connector), and the ALRT signal and I2C breakout to the compute board.

### Repo layout

```
EPS_V1/
├── EPS_V1.kicad_pro / .kicad_sch / .kicad_pcb / .step
├── fp-lib-table, sym-lib-table
├── schematics/          # sub-sheets
├── libs/
│   ├── footprints.pretty/
│   ├── symbols/
│   └── 3dmodels/
├── spice/
│   ├── sims/
│   ├── models/
│   └── translator/
├── tools/                # SPICE model translator script
└── manufacturing/
    └── rev-<N>_<date>/
        ├── gerbers/
        ├── bom/
        └── plots/
```

**Why 2S battery packs specifically:** 4x 2S (8 cells total) rather than one large pack — this is a mechanical/electrical packing choice under the ISS 80 Wh limit, and it gives pack-level redundancy (a single pack failure doesn't take down the whole EPS). It also matters electrically: a 2S pack has a ~6.0V floor, which the buck converters can cleanly step down to 5V and 3.3V — a single-cell pack's ~3.0V floor wouldn't support that.

## Key design notes

- **Charge voltage 4.2V, typical 3.6V, overdischarge floor raised from the cell's rated 2.75V to 3.0V** — added margin above the cell's absolute minimum.
- **MP2672A output current is being restricted below the MP28167GQ-Z's 3A capability** to stay under the board's actual current requirement (2.52A = 14/5×0.9). Open question below on whether that restriction should assume all 4 packs charging at once.
- **MAX17205's external-thermistor input is intentionally unused** — flight software reads temperature from the analog thermistor connection straight to the compute board instead, not through the fuel gauge.
- **TPS61088 (12V) key values:** 200kHz min switching frequency (840k resistor), 3A current limit (400k), PFM mode (MODE pin floating), R1 502k / R2 56k for 12V output, 2.2µH inductor per TI's recommended table.
- **Thermistor (NXFT15XV103FEAB025):** system draws 0.574mW through it against a 3mW rating — comfortable margin. Rated -40°C to 125°C.

## Simulations
- `spice/sims/` — EPS_U6, U2_EPS, U3_EPS, U6_EPS (regulator sims)
- `spice/models/` — TI PSpice models for TPS61088/TPS62132/TPS62133, both original and LTSpice-collapsed versions
- `spice/translator/` — script + requirements to collapse TI's multi-line `.lib` continuation format into something LTSpice's autogenerator can parse

## Manufacturing history
| Rev | Date | Notes | Location |
|-----|------|-------|----------|
| 1.0 | 5/15/26 | First layout pass | `manufacturing/rev-1.0_2026-05-15/` |
| 1.2 | 8/7/26  | Prepared for schematic submittal | `manufacturing/rev-1.2_2026-08-07/` |

## Open items / TBD
- S-band kill switch, FSW-configurable — not yet in design
- R_MID value for MP2672A — placeholder 0, needs datasheet-confirmed value (50Ω noted elsewhere from the fuel gauge datasheet — worth reconciling)
- Solar input voltage range (2.8–22V accepted) vs. actual solar board output — needs confirmation
- MP2672A max charge current restriction — should this account for a scenario where one pack is offline/broken?