# H2Sense Boards

KiCad hardware projects for a battery-powered hydrogen (H2) sensor node.
 
## SensorNode
 
The sensor node detects hydrogen using a surface acoustic wave (SAW) resonator as the sensing element. The resonator sets the frequency of an oscillator at about 110 MHz, and exposure to hydrogen shifts that frequency. The amount of shift indicates the H2 concentration.
 
To make the shift easy to measure, the 110 MHz signal (or lower, dipends on the device to measure) is mixed with a stable commercial 120 MHz (or 104MHz depending on the device to measure) reference. The resulting beat frequency, lower then 10 MHz, is filtered and squared up by a comparator. Its frequency can then be counted digitally, and any change in it follows the change in the sensor frequency.
 
Because the measurement relies on small frequency shifts, the stability and spectral purity of both oscillators are central to the design. This drives most of the choices on both boards: low-noise post-regulation, separate filtered supply branches, and keeping the switching converters off the RF board.

### Architecture

```
Battery ──► PowerBoard ──► 3.3 V / 5 V / −0.55 V ──► RF board

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

A separate power supply board that keeps the switching converters off the RF board. It is a test board: it provides footprints for two alternative buck converters, low-noise post-regulation with LDOs, a negative gate bias line, and adjustable test loads.

<img width="1158" height="797" alt="image" src="https://github.com/user-attachments/assets/906473ed-dc5c-4b74-ae58-934b50d4142d" />

<img width="1288" height="753" alt="image" src="https://github.com/user-attachments/assets/dcef96c4-8d7e-4a1b-9398-55946b6315c7" />


### Block diagram

```
J1 Battery (Vin)
   │
   ├──► U3  Dual buck module (Vin ≤ 6 V)   ┐   only one of the two
   │                                       ├── is populated
   └──► U6  LTC3607 dual buck (4.5–15 V)   ┘
              │
              ├── Buck_3V5 (~3.47 V) ──► U1 LDO 3.3 V ──► 3V3 ──► J4
              │                                           │
              │                                           └──► U4 TC7660 ──► −3V3
              │                                                               │
              │                         3V3 + −3V3 ──► U5 OPA191 ──► −0.55 V ─┴─► J2 (Vg)
              │
              └── Buck_5V3 (~5.31 V) ──► U2 LDO 5 V ──► 5V ──► J3, J4
```

### Input

| Parameter | Value |
|---|---|
| Connector | J1, 2-pin battery connector (XT30) |
| Input capacitor | C1, 22 µF |
| Input range | Up to 6 V with U3, 4.5–15 V with U6 |

A 2S Li-ion battery (6.0–8.4 V) requires U6. U3 can only be used with an input supply of about 5.5–6 V, since its 5.3 V output needs some headroom.

### Buck converters (populate only one)

U3 and U6 drive the same `Buck_3V5` and `Buck_5V3` nets. **Only one of them must be mounted.**

**U3: dual buck module (input up to 6 V)**

| Output | Feedback divider | Output voltage (VFB = 0.608 V) |
|---|---|---|
| Buck_3V5 (OUT1) | R4 = 100 kΩ, R5 = 21.3 kΩ | ≈ 3.46 V |
| Buck_5V3 (OUT2) | R3 = 100 kΩ, R6 = 13 kΩ | ≈ 5.28 V |

EN1 and EN2 are pulled up to Vin through R1 and R2 (100 kΩ). Output capacitors: C2, C3 (22 µF).

**U6: LTC3607 dual 600 mA synchronous buck (4.5–15 V)**

| Output | Inductor | Feedback divider | Output voltage (VFB = 0.6 V) |
|---|---|---|---|
| Buck_5V3 (SW1) | L1 = 4.7 µH | R23 = 1.82 MΩ, R21 = 232 kΩ, C13 = 22 pF | ≈ 5.31 V |
| Buck_3V5 (SW2) | L2 = 4.7 µH | R24 = 1.11 MΩ, R22 = 232 kΩ, C14 = 22 pF | ≈ 3.47 V |

- Output capacitors: C11, C12 (10 µF).
- RUN1 and RUN2 are tied to Vin, so both outputs start as soon as the battery is connected.
- PGOOD1 and PGOOD2 are pulled up to their outputs through R19 and R20 (100 kΩ).
- **J5** selects the operating mode on MODE/SYNC: pulse skipping (MODE = 0 V) or Burst Mode (MODE > 1 V). Burst Mode is more efficient at light load, pulse skipping gives a more predictable switching spectrum.

### Post-regulation

| Rail | Regulator | Input | Capacitors |
|---|---|---|---|
| 3V3 | U1, LDO 3.3 V (TLV70233, SOT-23-5) | Buck_3V5 | C6 = 1 µF in, C5 = 1 µF out |
| 5V | U2, LDO 5 V | Buck_5V3 | C7 = 2.2 µF in, C4 = 1 µF out |
| -0.55V | U5, OpAmp | Buck_5V3 | Cf = 1 µF in, C35 = 0.1 µF out |
| 0.5V | U5, OpAmp | Buck_5V3 | Cf = 1 µF in, C33 = 0.1 µF out |

The bucks run about 0.2–0.3 V above the LDO outputs, so the LDOs dissipate very little while filtering the switching ripple.

### −0.55 V gate bias line

| Part | Function |
|---|---|
| U4, TC7660 | Inverting charge pump: 3V3 → −3V3 (C8 = 10 µF flying, C9 = 10 µF output, LV pin grounded) |
| R9 + C10 | RC filter (1 kΩ, 1 µF) on the op-amp negative supply |
| U5, OPA191 | Inverting amplifier supplied from 3V3 / −3V3, non-inverting input grounded |
| R7 | Input resistor from 3V3 (100 kΩ) |
| R8 + R16 | Fixed feedback: 16.5 kΩ, giving Vout ≈ −3.3 V × 16.5k / 100k ≈ −0.545 V |
| R12 + R17 | Alternative adjustable feedback: 20 kΩ trimmer (Vishay T93YA), range 0 to about −0.66 V |
| J2 | Output connector (Vg) |

Populate **either** R16 (fixed value) **or** R17 (trimmer) as a 0 Ω jumper.

This line is optional. The SKY65050 datasheet recommends self-biasing instead: a source resistor (27–47 Ω, RF-bypassed) with the gate held at 0 V DC, which removes the negative supply and the need for power sequencing.

If the negative line is used on the RF board:

- the gate voltage must be applied **before** the drain supply and removed **after** it;
- the gate current must stay below 100 µA (absolute maximum).

### Outputs

| Connector | Signals |
|---|---|
| J4 (3-pin header) | 5V, 3V3, GND |
| J3 | 5V |
| J2 | −0.55 V (Vg) |
| J6 | 0.5 V (V_heater) |

### Test loads

| Rail | Load | Notes |
|---|---|---|
| 5V | R13 (0 Ω) + R11 (20 kΩ trimmer, rheostat) | Adjustable load to GND |
| 3V3 | R14 (0 Ω) + R10 (20 kΩ trimmer, rheostat) | Adjustable load to GND |
| −0.55 V | R15 (0 Ω) + R18 (232 kΩ) | Fixed load, about 2.4 µA |
| 0.5 V | none | No test load for heaters power supply |

The 0 Ω resistors disconnect each test load when not mounted.

---

## RF board

| Block | Part / notes |
|---|---|
| 104 MHz oscillator | Pierce with SKY65050-372LF pHEMT and SAW resonator |
| LO oscillator | Commercial 120 MHz oscillator |
| Mixer | 104 MHz on the IF port, 103 MHz on the LO port, ~1 MHz output |
| Output filter | Low-pass / band-pass around 10 MHz |
| Comparator | Supplied from 5 V and 3.3 V, output on SMA_out |

> **Note:** the SKY65050-372LF is discontinued. Consider a replacement for future revisions.

### Design notes

- Keep the commercial oscillator physically separated from the comparator section and powered separately with filts to avoid mutual pulling and spurs.
- The FET has gain up to several GHz: add a small series resistor on the gate or a ferrite bead on the drain to prevent parasitic oscillations, and check the spectrum up to a few GHz.
- Use a high-value gate DC return (100 kΩ – 1 MΩ) to avoid lowering the resonator Q.
- Use C0G/NP0 capacitors in the Pierce feedback network.
- Continuous ground plane under the whole RF section, with stitching vias and local decoupling (100 nF + nF) on every IC.

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
