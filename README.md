# SoC Design of PicoRV32 RISC-V Micro-Processor

This repository documents my journey through the VSD SoC Design and Planning 
Workshop — implementing a complete Physical Design flow for the PicoRV32 
RISC-V processor using open-source tools.

**Workshop:** VSD SoC Design and Planning  
**Tools:** OpenLANE | Magic | OpenSTA | NGSpice | Sky130 PDK  
**Design:** PicoRV32 RISC-V Core  

---

## Day 1 — Getting Started with Open-Source Chip Design

### The Chip vs The Package

Most people point to the black component on a circuit board and call it 
a chip — but that's actually the **package**. The real silicon die sits 
hidden inside, protected by this casing. It connects to the outside world 
through **wire bonds** — microscopic wires that link the die's contact 
pads to the package pins.

### What's Inside a Chip

Zooming into the silicon die, three regions define its structure:

- **Core** — where all the logic gates, flip flops and digital circuits live
- **Pads** — the boundary ring through which signals enter and exit the chip
- **Die** — the complete silicon piece that gets cut from the wafer

A few terms worth knowing:
- **Foundry** — the factory that physically manufactures the chip (eg. TSMC, SkyWater)
- **Foundry IPs** — blocks like PLLs and SRAMs that only the foundry knows how to build properly
- **Macros** — pre-designed digital blocks that get dropped into the layout

### How Software Becomes Silicon

When you write code in C and run it on a chip, it travels through 
multiple layers before it becomes electrical signals:
C Program

↓ Compiler

RISC-V Assembly

↓ Assembler

Binary (0s and 1s)

↓ RTL Implementation

Hardware Logic

↓ Physical Design Flow

Silicon Chip
The OS, compiler and assembler together form the bridge between 
human-written code and the actual hardware that executes it.

### The Open-Source ASIC Design Stack

Three ingredients are needed to design a chip:

| Ingredient | What it is | Where to get it |
|---|---|---|
| RTL Design | Hardware description of the chip | opencores.org, librecores.org |
| EDA Tools | Software to implement the design | OpenLANE, Magic, OpenSTA |
| PDK Data | Foundry rules and cell libraries | Sky130 PDK by Google + SkyWater |

Before 2020, PDKs were only available under NDAs — meaning chip design 
was locked behind closed doors. In **June 2020**, Google and SkyWater 
changed everything by releasing **Sky130** as the world's first open-source 
PDK. This made real chip design accessible to students and researchers 
for the first time.

### OpenLANE — The Full Flow in One Tool

| Stage | Tool | What Happens |
|---|---|---|
| Synthesis | Yosys + ABC | RTL code is converted into logic gates using standard cell library |
| Floorplanning | OpenROAD | Chip size is defined, blocks are positioned, power grid is created |
| Placement | OpenROAD | Standard cells are placed into rows inside the chip area |
| Clock Tree Synthesis | TritonCTS | Clock signal is distributed to all flip flops with minimum skew |
| Routing | FastRoute + TritonRoute | Metal wires connect all placed cells following PDK design rules |
| Parasitic Extraction | OpenRCX | Wire resistance and capacitance are extracted for accurate timing |
| Layout Export | Magic + KLayout | Final layout is exported as GDSII file for fabrication |
| Timing Sign-off | OpenSTA | Setup and hold timing is verified across all paths |
| Physical Verification | Magic + Netgen | DRC and LVS checks confirm layout is manufacture-ready |

## Lab 1 — OpenLANE Interactive Flow and Synthesis of picorv32a

### Running the Flow

OpenLANE is launched in interactive mode and the picorv32a design 
is prepared and synthesized step by step. The screenshot below shows 
all the commands run in sequence — mounting OpenLANE, launching 
interactive flow, preparing the design and running synthesis.

```bash
cd ~/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
prep -design picorv32a
run_synthesis
```

![Total cell count](images/01_syn1.png)

### Synthesis Report — Cell Count

After synthesis we open the report to check the total number 
of cells in the design — **15762 cells** in total.

![Total cell count](images/02_syn2.png)

### Synthesis Report — Flip Flop Count

From the same report we identify the DFF count — **1613 flip flops** 
used in the design.

![DFF count](images/03_syn3.png)

### Flop Ratio Calculation
Flop Ratio = 1613 / 15762 = 10.23%

10% of the total cells are flip flops — healthy for a 
processor design like PicoRV32.

# Day 2 — Good FloorPlan vs Bad FloorPlan and Introduction to Library Cells

---

## 1. Chip Floorplanning

Floorplanning is the first step after synthesis. It decides the size and shape of the chip, where IO pins go, and where big blocks (like memories) are placed.

Two key parameters:
- **Utilisation Factor** = Area used by cells / Total core area. Typical value: 0.5–0.6 (leave room for routing and buffers)
- **Aspect Ratio** = Height / Width. A value of 1 means a square chip.

---

## 2. Pre-Placed Cells and Decoupling Capacitors

**Pre-placed cells** are large blocks (memories, PLLs, IPs) that are placed manually before the tool runs. The tool won't move them.

**Decoupling capacitors (DECAPs)** are placed around these blocks. When many gates switch at once, they need a lot of current instantly. DECAPs act like local batteries — they supply that current and prevent the voltage from dropping, which could cause logic errors.

---

## 3. Power Planning

Power is delivered using a **power mesh** — a grid of VDD and VSS wires on upper metal layers covering the whole chip. This ensures every cell gets power from nearby, reducing **IR drop** (voltage loss due to wire resistance) and **electromigration** (wire damage from high current).

---

## 4. Pin Placement and Cell Blockage

IO pins are placed along the chip edges. Pins are placed close to the logic they connect to, so wires stay short.

The area between the core and the die edge is blocked — called a **logical cell blockage** — so the placer doesn't put standard cells there. That space is reserved for IO buffers and ESD cells.

---

## 5. Library Binding and Placement

Each gate in the netlist is mapped to a real physical cell from the library. The library has cells of different sizes and speeds for the same logic function.

- **Initial placement** puts cells inside the core, trying to keep connected cells close together.
- **Placement optimisation** checks wire lengths. If a wire is too long and signal quality drops, **buffers (repeaters)** are inserted to fix it.

---

## 6. Cell Design Flow

Every standard cell is created through this flow:

**Inputs** → PDK (DRC/LVS rules, SPICE models, design rules)

**Design Steps:**
- **Circuit design** — size the transistors to meet speed and drive strength targets
- **Layout design** — draw the physical layout using Euler's path + stick diagram to minimise diffusion breaks
- **Characterisation** — simulate the cell to extract timing, power, and noise data

**Outputs** → CDL netlist, GDSII, LEF, extracted SPICE (.cir), timing/power/noise `.lib` files

Cells come in different flavours: different functions (AND, OR, buffer, DFF...) and different threshold voltages (hvt, svt, lvt) to trade off speed vs power.

---

## 7. Characterization Flow

The **GUNA** tool takes 8 inputs and generates the `.lib` models used in STA:

1. NMOS and PMOS SPICE models
2. Extracted cell subcircuit (`.sub` file)
3. Testbench — two cascaded inverters with a 10f load cap
4. Cell subcircuit (PMOS: W=0.9u/L=0.18u, NMOS: W=0.36u/L=0.18u)
5. Power supply (VDD = 1.8V)
6. Input pulse stimulus
7. Output load capacitance
8. Transient simulation command (`.tran 10e-12 4e-09`)

GUNA outputs four model types: **Timing**, **Noise**, **Power**, and **Function** — all stored as `.lib` files used by STA and PD tools.

---

## 8. Timing Characterization

Timing is measured using 8 voltage threshold variables defined in the `.lib` file:

- `slew_low_rise_thr`, `slew_high_rise_thr` — low and high thresholds for rise slew
- `slew_low_fall_thr`, `slew_high_fall_thr` — low and high thresholds for fall slew
- `in_rise_thr`, `in_fall_thr` — input thresholds for delay measurement
- `out_rise_thr`, `out_fall_thr` — output thresholds for delay measurement

The actual threshold values depend on the PDK and the cell being characterised.

**Propagation delay** = `time(out_*_thr) − time(in_*_thr)`
- A well-chosen threshold gives a small positive delay
- A bad threshold choice or too large an output load can give a negative delay — this is incorrect and must be avoided

**Transition time (slew):**
- Rise slew = `time(slew_high_rise_thr) − time(slew_low_rise_thr)`
- Fall slew = `time(slew_high_fall_thr) − time(slew_low_fall_thr)`

---

## 9. Lab — Floorplan

### Running Floorplan

```bash
run_floorplan
cd results/floorplan/
less picorv32a.def
```

### Floorplan DEF File

The output DEF file is at:
```
runs/RUN_2026.06.17_18.05.17/results/floorplan/picorv32a.def
```

![Floorplan terminal — steps 1 to 6](images/04_floorplan1.png )

![DEF file — die area and row definitions](images/05_floorplan2.png)
picorv32a floorplan DEF file — DIEAREA and standard cell ROW definitions


### Floorplan in Magic

```bash
magic -T /home/vscode/.ciel/ciel/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.tech \
  lef read ../../tmp/merged.nom.lef \
  def read picorv32a.def &
```

![picorv32a floorplan in Magic](images/06_floorplan3.png)
picorv32a floorplan in Magic — purple/pink = pwell, cyan lines = standard cell rows, IO pins on all 4 edges

![Zoomed floorplan in Magic — IO pins and tap cells](images/07_floorplan4.png)



---

## 10. Lab — Placement

### Running Placement

```bash
run_placement
```

![OpenLANE terminal — placement flow](images/08_placement1.png)
Terminal showing all placement steps completing successfully for picorv32a

### Placement in Magic

```bash

cd /home/vscode/Desktop/OpenLane/designs/picorv32a/runs/RUN_2026.06.17_18.05.17/results/placement/
magic -T /home/vscode/.ciel/ciel/sky130/versions/0fe599b2afb6708d281543108caf8310912f54af/sky130A/libs.tech/magic/sky130A.tech \
  lef read ../../tmp/merged.nom.lef \
  def read picorv32a.def &
```

**Global placement view** — all standard cells are placed inside the core. The dense dark regions are clusters of logic cells.


![picorv32a global placement in Magic](images/09_placement2.png)
*picorv32a after global placement — standard cells distributed across the core*

**Zoomed-in view** — individual `sky130_fd_sc_hd_*` standard cells are visible. The horizontal striped bands are the **VPWR** and **VGND** power rails running across each cell row.

![Zoomed placement view — standard cells and power rails](images/10_placement3.png)

# Day 3 — Design and Characterisation of Library Cells using Magic & ngspice

## CMOS Inverter — SPICE Deck

To characterise a standard cell, we write a SPICE netlist describing the PMOS and NMOS transistors along with their W/L ratios, supply voltage, input stimulus, and load capacitance.

```
M1 out in vdd vdd pmos W=0.375u L=0.25u
M2 out in 0 0 nmos W=0.375u L=0.25u
cload out 0 10f
Vdd vdd 0 2.5
Vin in 0 2.5
.op
.dc Vin 0 2.5 0.05
.LIB "tsmc_025um_model.mod" CMOS_MODELS
.end
```

Key parameters extracted from simulation:

- **Rise time** — output rising edge, 20% → 80%
- **Fall time** — output falling edge, 80% → 20%
- **Propagation delay** — 50% input → 50% output

Also characterised the switching threshold **Vm** (where Vout = Vin). With Wn = Wp = 0.375µm, Vm ≈ 0.98V — the NMOS pulls down harder since electron mobility beats hole mobility. Widening the PMOS to ~2.5× the NMOS width (0.9375µm) brings Vm closer to the ideal VDD/2, landing at ≈1.2V on a 2.5V supply. This is also why PMOS is drawn wider than NMOS in real standard cell libraries — a skewed Vm means asymmetric noise margins.

## 16-Mask CMOS Fabrication Process (Brief Overview)

Chip fabrication follows a sequence of about 16 mask steps. Steps 1–10 build the transistors themselves (front-end), and steps 11–16 wire them together with interconnect (back-end).

1. Substrate selection (p-type, high resistivity)
2. Active region formation (isolation via LOCOS/STI, field oxidation + Si3N4 mask)
3. N-well formation (ion implantation)
4. P-well formation (ion implantation)
5. Gate oxide growth
6. Polysilicon gate deposition and patterning
7. Lightly Doped Drain (LDD) formation
8. Sidewall spacer formation
9. N+ source/drain implantation (with halo implants)
10. P+ source/drain implantation (with halo implants)
11. Local interconnect / silicidation
12. Contact formation
13. Metal 1 deposition and patterning
14. Via formation
15. Higher-level metal formation (Metal 2 and above)
16. Final passivation

## Lab — Cloning and Characterising a Custom Inverter Cell

Cloned the standard cell repository and opened the inverter layout in Magic:

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
magic -T sky130A.tech sky130_inv.mag &
```

Inspected the cell: stacked PMOS/NMOS pair, VPWR/VGND rails, and A (input) / Y (output) ports.

![sky130_inv layout in Magic](images/day3/11_inv.png)
![sky130_inv layout in Magic](images/day3/12_inv2.png)
![sky130_inv layout in Magic](images/day3/13_inv3.png)
![sky130_inv layout in Magic](images/day3/14_inv4.png)
![sky130_inv layout in Magic](images/day3/15_inv5.png)

## Extracting SPICE Netlist from Magic

From the tkcon console:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

This produces `sky130_inv.spice` — a netlist built straight from the drawn layout geometry rather than hand-written. Edited the generated file to check the model include path and unit distances before simulating.

![sky130_inv layout in Magic](images/day3/inv_layout.png)

## Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```
```
plot y vs time a
```

![ngspice transient waveform - output vs input](images/day3/ngspice_waveform.png)

From the waveform, measured rise time, fall time, and propagation delay:

**Rise transition time** = time to reach 80% − time to reach 20%
- 20% of output = 660 mV
- 80% of output = 2.64 V

**Fall transition time** = time to fall to 20% − time to fall to 80%
- 20% of output = 660 mV
- 80% of output = 2.64 V

Propagation delay measured at the input/output 50% crossover, around **t ≈ 2.18 ns**.

![Zoomed-in crossover point for delay measurement](images/day3/ngspice_delay_zoom.png)

## Magic DRC Lab

Reference: [Sky130 Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html)

**Poly rule (poly.9):** found a case where poly.9 was incorrectly implemented in the old sky130A tech file — spacing under 0.48µm wasn't flagging a DRC violation at all. Traced this to the rule definition in the tech file itself and corrected it so the spacing check actually fires.

![poly.9 rule before correction - no violation flagged](images/day3/poly9_before.png)

**N-well:**
```tcl
% drc why
N-well width < 0.84um (nwell.1)
N-well spacing < 1.27um (nwell.2a)
N-well overlap of Deep N-well < 0.4um outside, 1.03um inside (nwell.5a, 7)
```
Fixed the geometry — follow-up `drc why` returned "No errors found."

![N-well DRC violations and corrected cell](images/day3/nwell_drc.png)

**Diffusion tap (difftap):** worked through `difftap.1`–`difftap.6`, comparing incorrect examples against corrected versions in the same layout.

![difftap correct vs incorrect examples](images/day3/difftap_examples.png)

**Poly / precision resistor:**
```tcl
% drc why
mrp1 resistor width < 0.33um (poly.3)
xhrpoly/uhrpoly resistor spacing to diffusion < 0.48um (poly.9)
poly.resistor spacing to N-tap < 0.48um (poly.9)
poly.spacing to Diffusion < 0.075um (poly.4a)
P-tap spacing to field poly < 0.055um (poly.5)
```
Worked through `poly.1a`–`poly.16`, each labeled Correct by design / Incorrect / Not implemented.

![poly.1a to poly.16 test structures](images/day3/poly_test_structures.png)

*Note: a cosmetic grey-crosshatch rendering issue in this Codespaces + noVNC setup doesn't affect DRC accuracy.*

---
*Part of the [SoC Design of the PicoRV32 RISC-V micro-processor - VSD](https://github.com/saisindhumanne34/SoC-Design-of-the-PicoRV32-RISCV-micro-processor-VSD) workshop series.*



