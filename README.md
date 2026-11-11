# PCBprojects

A collection of hardware projects designed with KiCad.

## SensorNode

A radio-frequency system built around an oscillator using a surface acoustic wave (SAW) resonator at about 110 MHz. The oscillator signal is mixed with a commercial 120 MHz reference. The resulting beat frequency, around 10 MHz, is filtered and squared up by a comparator so its frequency can be measured digitally.

The system is split into two boards: an RF board and a power board (PowerBoard).

### Architecture

```
2S Li-ion battery ──► PowerBoard ──► 3.3 V / 5 V ──► RF board

RF board:
  Pierce oscillator + SAW (~110 MHz) ──► IF port ┐
                                                 ├─ Mixer ──► Filter (~10 MHz) ──► Comparator ──► SMA_out
  Commercial 120 MHz oscillator ──────► LO port  ┘
```

### Repository structure

```
PCBprojects/
├── .gitignore
├── README.md
└── SensorNode/
    └── PowerBoard/      Power supply board (KiCad project)
```

The RF board will be added in its own subfolder of `SensorNode/`.

---

## PowerBoard

A separate power supply board that keeps the switching converters off the RF board. It is designed for low power consumption and, in the test version, allows different regulation configurations to be compared.

### Input

| Parameter | Value |
|---|---|
| Battery | 2S Li-ion (6.0–8.4 V, 7.4 V nominal) |
| Connector | XT30, PCB-mount version |
| Protection | Reverse polarity (P-MOSFET), fuse, undervoltage cutoff (around 6.0–6.2 V) |

### Outputs

| Output | Loads | Estimated current | Regulation |
|---|---|---|---|
| 3.3 V | 120 MHz oscillator, comparator (3.3 V side) | ~45 mA | Buck + TLV70233 LDO, or buck only |
| 5 V | Comparator (5 V side) | ~5 mA | Buck + LDO, or LDO from battery |
| −0.55 V (optional) | Gate of the SKY65050 FET | ~10 µA | Inverting charge pump + inverting op-amp |

### Test configurations

The test board allows comparing:

- **Buck + LDO vs. buck only:** selected with 0 Ω resistors. The buck output voltage must be changed accordingly: about 3.5–3.6 V with the LDO, 3.3 V without it.
- **PFM vs. forced PWM mode:** jumper on the buck MODE pin.
- **LC filter:** configurable footprints, with an optional second stage.

The board includes shunts for efficiency measurements, ripple test points with short ground connections, and footprints for dummy load resistors.

> **Configuration note:** before connecting the RF board, check that the buck output voltage matches the selected configuration. In buck-only mode, a 3.5 V setting would go straight to the RF board.

### −0.55 V line

Negative gate bias is optional. The alternative recommended in the SKY65050 datasheet is self-biasing: a source resistor (27–47 Ω, RF-bypassed) with the gate held at 0 V DC. This removes the need for a negative supply and for power sequencing.

If the negative line is used:

- the gate voltage must be applied **before** the drain supply and removed **after** it;
- the gate current must not exceed 100 µA (absolute maximum): use a series resistor of at least 27–33 kΩ or a clamp diode;
- the charge pump ripple must be filtered with RC networks on the op-amp negative supply and on the output.

### Components

| Function | Part | Status |
|---|---|---|
| 3.3 V LDO | TI TLV70233 (SOT-23-5) | Selected |
| 3.3 V / 5 V buck | — | TBD (input ≥ 9 V required for 2S) |
| 5 V LDO | — | TBD |
| Inverting charge pump | — | TBD |
| Op-amp for −0.55 V | LT6003 | Candidate |
| Battery connector | Amass XT30 (UPB or PW) | Selected |

---

## RF board

| Block | Part / notes |
|---|---|
| 110 MHz oscillator | Pierce with SKY65050-372LF pHEMT and SAW resonator |
| LO oscillator | Commercial 120 MHz oscillator |
| Mixer | 110 MHz on the IF port, 120 MHz on the LO port, ~10 MHz output |
| Output filter | Low-pass / band-pass around 10 MHz |
| Comparator | Supplied from 5 V and 3.3 V, output on SMA_out |

> **Note:** the SKY65050-372LF is discontinued. Consider a replacement for future revisions.

### Design notes

- Keep the two oscillators (110 and 120 MHz) physically separated and powered from separately filtered branches to avoid mutual pulling and spurs.
- The FET has gain up to several GHz: add a small series resistor on the gate or a ferrite bead on the drain to prevent parasitic oscillations, and check the spectrum up to a few GHz.
- Use a high-value gate DC return (100 kΩ – 1 MΩ) to avoid lowering the resonator Q.
- Use C0G/NP0 capacitors in the Pierce feedback network.
- Continuous ground plane under the whole RF section, with stitching vias and local decoupling (100 nF + nF) on every IC.

---

## Estimated power budget

| Item | Power |
|---|---|
| 120 MHz oscillator (3.3 V × 40 mA) | ~0.13 W |
| Comparator | ~0.04 W |
| SAW oscillator with SKY65050 (~3 V × 10–20 mA) | ~0.03–0.06 W |
| Regulation losses | ~0.05–0.1 W |

Indicative values, to be updated with measurements on the test board.

---

## Working on the project

### Requirements

- KiCad (state the version used by the team: files saved with a newer version cannot be opened with an older one)
- Git

### Cloning the repository

```
git clone https://github.com/GiacomoGobbo/PCBprojects.git
```

### Workflow

1. Before starting: `git pull`.
2. Work on the project in KiCad.
3. When done:
   ```
   git add .
   git commit -m "Description of the changes"
   git push
   ```

### Rules

- Do not edit the same `.kicad_sch` or `.kicad_pcb` file at the same time: conflicts in KiCad files are hard to resolve. Split the work by board or by schematic sheet.
- Keep custom symbol, footprint and 3D model libraries inside the project folder, using relative paths (`${KIPRJMOD}`).
- The XT30 footprint is in the official `Connector_AMASS` library. Use the `Conn_01x02` symbol in the schematic.
- Do not create nested Git repositories inside project subfolders.
- The `.gitignore` in the root folder excludes backups, autosaves, caches, lock files, local preferences (`.kicad_prl`) and KiCad's local history (`.history/`).

---

## Authors

- Giacomo Gobbo
