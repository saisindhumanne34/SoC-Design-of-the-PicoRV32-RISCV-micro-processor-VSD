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

![sky130_inv layout - full cell view](images/11_inv.png)
![sky130_inv layout - pmos layer selected](images/12_inv2.png)
![sky130_inv layout - nmos layer selected](images/13_inv3.png)
![sky130_inv layout - with DRC layer legend](images/14_inv4.png)
![sky130_inv layout - with routing layer legend](images/15_inv5.png)

## Extracting SPICE Netlist from Magic

From the tkcon console:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```
Measuring unit distance in layout grid

![Extracted SPICE file - tkcon console showing extract/ext2spice commands](images/16_inv6.png)

## Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```
```
plot y vs time a
```

![ngspice terminal output - node voltages](images/18_inv8.png)
![ngspice terminal output - with plot commands and delay measurements](images/19_inv19.png)

![ngspice transient waveform - output vs input](images/17_inv7.png)

From the waveform, measure rise time, fall time, and propagation delay values.

**Rise transition time calculation**

Rise transition time = Time taken for output to rise to 80% − Time taken for output to rise to 20%
- 20% of output = 660 mV
- 80% of output = 2.64 V

**Fall transition time calculation**

Fall transition time = Time taken for output to fall to 20% − Time taken for output to fall to 80%
- 20% of output = 660 mV
- 80% of output = 2.64 V

Propagation delay measured at the input/output 50% crossover, around **t ≈ 2.18 ns**.

![Zoomed crossover - 2.56-2.72V range](images/22_inv11.png)
![Delay measurement values from ngspice cursor](images/19_inv9.png)
![Zoomed crossover - 665.4-667.4mV range](images/20_inv10.png)
![Zoomed crossover - 610-710mV range](images/21_inv11.png)

## Magic DRC Lab

Reference: [Sky130 Periphery Rules](https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html)

![Sky130 PDK periphery rules documentation](images/23_sky1.png)

**Met3:** worked through a similar labeled test grid (`m3.1`–`m3.6`) for the met3 layer, again split into correct-by-design, incorrect, and not-implemented examples.

![met3 test structures - m3.1 to m3.6](images/24_sky2.png)
![poly cell zoomed - ppolyres layer, DRC=22](images/25_sky3.png)

**Poly rule (poly.9):** found a case where poly.9 was incorrectly implemented in the old sky130A tech file — spacing under 0.48µm wasn't flagging a DRC violation at all. Traced this to the rule definition in the tech file itself and corrected it so the spacing check actually fires.

![poly.9 rule - drc why output showing mrp1/poly.9 violations](images/27_sky5.png)

**N-well:**
```tcl
% drc why
N-well width < 0.84um (nwell.1)
N-well spacing < 1.27um (nwell.2a)
N-well overlap of Deep N-well < 0.4um outside, 1.03um inside (nwell.5a, 7)
```
Fixed the geometry — follow-up `drc why` returned "No errors found."

![N-well - drc why output showing width/spacing/overlap violations](images/30_sky8.png)
![N-well - incorrect implementation flagged, DRC=24](images/29_sky7.png)
![N-well - polysilicon zoomed view, DRC=11](images/31_sky9.png)

**Diffusion tap (difftap):** worked through `difftap.1`–`difftap.6`, comparing incorrect examples against corrected versions in the same layout.

![difftap.1 to difftap.6 - correct vs incorrect, DRC=10](images/32_sky10.png)

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

![poly.1a to poly.16 test structures](images/26_sky4.png)
![poly.7-poly.11 structures with xhrpoly/uhrpoly drc why output](images/28_sky6.png)

# Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

## LEF Files and Guidelines for Standard Cell Ports

A custom cell can't be dropped into OpenLANE without a LEF file describing its outline, pin positions, and metal geometry. Two placement rules matter most for the router to actually reach the pins:

- Ports need to land exactly where a horizontal and vertical routing track cross
- The cell's width and height each have to be an odd multiple of the corresponding track pitch

## Clock Gating

A simple technique to save power by cutting the clock to idle logic.

**AND gate:** EN=1 → clock passes → circuit active. EN=0 → clock blocked → circuit sleeps → saves power.

**OR gate:** EN=0 → clock passes → circuit active. EN=1 → output stuck at 1 → clock blocked → saves power.

Either way: block the clock = save power.

## Delay Tables

Buffer delay isn't fixed — it depends on input slew (how fast the signal switches) and output load capacitance (how much is connected at the output). Rather than recalculating this every time, all combinations are pre-characterized into a 2D lookup table that the tool reads during CTS: rows for input slew, columns for output load, interpolating between the nearest values if the exact combination isn't in the table.

## Static Timing Analysis (STA) Concepts

A path passes setup checking when there's slack left over: **slack = (time data is required by) − (time data actually arrives)**, and that number has to stay non-negative.

**Building up the setup condition:**
- Start simple: the combinational delay θ just has to fit inside one clock period T → θ < T
- A flop can't capture instantly — it needs a bit of settling time S before the edge, so the usable window shrinks: θ < T − S
- Real clocks don't land at the exact same instant every cycle (jitter), so a margin SU gets carved out too: θ < T − S − SU

**Two more variation sources STA has to absorb:**
- **OCV** — die-to-die and even within-die process/voltage/temperature swings, handled by applying derating factors to path delays rather than assuming one fixed number everywhere
- **CRPR** — when the launch and capture paths share a stretch of common clock buffering, that shared delay shouldn't be counted as extra pessimism on both sides; CRPR strips that double-count back out

## Clock Tree Synthesis (CTS)

After placement, flip-flops sit on the chip but the clock isn't connected yet. CTS builds the clock distribution network so it reaches every flop at the same time with minimum skew.

**Without CTS:** connecting the clock directly from source to every flop means far-away flops get it later than nearby ones — that arrival-time difference is skew, and it causes timing violations.

**H-Tree routing:** the tool finds the midpoint of all flip-flops and routes the clock in an H-shaped pattern, giving equal wire length — and therefore equal delay — from source to every flop.

**Buffering:** long clock wires have enough resistance and capacitance to degrade the signal, so buffers are inserted along the clock path to keep transitions sharp.

**Two things need a second look once CTS finishes:**
- **Hold timing** — the buffers CTS just inserted add real, physical delay to clock paths that were previously treated as ideal (zero-delay), and that shift can open up hold violations that weren't visible before
- **Setup timing** — worth re-running too, since the clock arrival times going into the setup calculation have changed from what was assumed pre-CTS

## Crosstalk

When two wires run close together, a switching signal on one (the aggressor) induces noise on the neighboring wire (the victim) through mutual capacitance.

**Glitch:** the aggressor's switching couples a false pulse onto the victim net — if that net is something like RST, the glitch can trigger an accidental reset and wipe valid data.

**Delta delay / skew:** crosstalk can also just add a small extra delay (Δ) to the victim net instead of a full glitch. In a clock tree, two paths designed to be equal for zero skew end up with skew = Δ once one path picks up coupling.

**Solution — clock net shielding:** shield wires tied to VDD/GND run on both sides of the clock net, absorbing switching noise before it reaches the clock signal.

---

## Lab — Custom Cell Integration and STA with OpenSTA

Checked `tracks.info` for the sky130_fd_sc_hd library to get the routing track pitch values needed for LEF port placement:

![tracks.info - li1/met1-met5 track pitch values](images/33_day4.png)

Used `help grid` to get the grid command syntax, then set the grid to match the track pitch:
```tcl
grid 0.46um 0.34um 0.23um 0.17um
```

![Magic grid set to track pitch, box command output](images/36_day4.png)
![sky130_inv cell aligned to grid - A/Y ports on track intersections](images/35_day4.png)
![sky130_inv full cell view on grid](images/34_day4.png)

Generated the LEF file from the tkcon console:
```tcl
lef write
```

![lef write command - generating sky130_vsdinv.lef](images/37_day4.png)

The resulting LEF defines the macro's boundary, site, and pin geometry:

![sky130_vsdinv.lef contents - PIN A/Y/VPWR definitions](images/38_day4.png)
![sky130_vsdinv.lef contents continued](images/40_sky4.png)

Copied the newly generated LEF and the required `.lib` files into the `picorv32a` design's `src` directory:

![src directory listing - sky130_vsdinv.lef and .lib files copied](images/39_day4.png)

### Editing `config.tcl` to Include the Custom Cell

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

Kicked off the flow, which picked up and merged the custom LEF automatically:

![OpenLane flow.tcl -interactive - merging sky130_vsdinv.lef, run_synthesis](images/41_day4.png)

Checked the placement DEF in Magic to confirm the custom inverter sits properly abutted against the standard cells around it:
![Internal connectivity layers - expand command view](images/42_day4.png)

![Custom inverter abutted in placement DEF among standard cells](images/44_day4.jpeg)
![Placement DEF - custom inverter cell instance with neighboring std cells](images/43_day4.jpeg)

### Running OpenSTA (Pre-CTS Timing)

Created `pre_sta.conf` in the OpenLane root, and `my_base.sdc` in `openlane/designs/picorv32a/src` (based on `openlane/scripts/base.sdc`), then ran:
```tcl
sta pre_sta.conf
```
![OpenLane flow.tcl -interactive - merging sky130_vsdinv.lef, run_synthesis](images/45_day4.png)
![OpenLane flow.tcl -interactive - merging sky130_vsdinv.lef, run_synthesis](images/46_day4.png)
![OpenLane flow.tcl -interactive - merging sky130_vsdinv.lef, run_synthesis](images/47_day4.png)


### Running CTS

```tcl
run_cts
```
![OpenLane flow.tcl -interactive - merging sky130_vsdinv.lef, run_synthesis](images/48_day4.png)

Dropped into the OpenROAD tool directly to build a custom timing report, reading the post-CTS netlist, liberty files, and the custom SDC:
```tcl
openroad
read_lef /OpenLane/designs/picorv32a/runs/<run_tag>/tmp/merged.nom.lef
read_def /OpenLane/designs/picorv32a/runs/<run_tag>/results/cts/picorv32a.def
write_db pico_cts.db
read_db pico_cts.db
read_verilog /OpenLane/designs/picorv32a/runs/<run_tag>/results/synthesis/picorv32a.v
read_liberty $::env(LIB_SYNTH_COMPLETE)
link_design picorv32a
read_sdc /OpenLane/designs/picorv32a/src/my_base.sdc
set_propagated_clock [all_clocks]
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4
```
# Day 5 — Final RTL to GDSII using TritonRoute & OpenSTA

## Routing — Global vs Detailed

Routing happens in two passes, each handing off to the next:

- **Global routing (FastRoute)** — splits the chip into a grid of routing regions and works out a rough path for every net through that grid, staying aware of layer usage and congestion but not committing to exact wire geometry yet.
- **Detailed routing (TritonRoute)** — takes those global routing guides and turns them into actual wire segments, vias, and track assignments, this time enforcing full DRC compliance.

### How global routing finds a path

At its core, global routing is a maze-search problem. The classic approach (Lee's algorithm, 1961) floods outward from the source cell in expanding rings, numbering each reachable grid cell by how many hops it took to get there, until the wave reaches the target. Backtracking from the target through decreasing numbers traces out a shortest path. FastRoute builds on this idea over a 3D grid, where each layer is broken into **global cells** (small regions of the chip) connected by **global edges** — and it's the capacity of those edges that global routing is really optimizing against, not individual wires yet.

Before a net's topology is even decided, there's a separate optimization step: given a set of access points to connect, build a minimum spanning tree over the pairwise distances between them. That MST becomes the topology the router then tries to realize — it's what keeps total wirelength down before any actual routing happens.

### How TritonRoute uses the guides it's handed

TritonRoute doesn't route freely — it treats the global routing guides as a strong hint and tries to stay inside them wherever possible. Before routing starts, those raw guides get cleaned up into a usable form:

- **Splitting** — breaking an oddly-shaped guide into unit-width, single-direction pieces
- **Merging** — combining adjacent pieces that run the same direction
- **Bridging** — adding short connecting segments where two guides on different layers need to overlap to actually connect

The output has to satisfy two conditions: every guide is unit-width, and every guide runs in that layer's preferred direction (e.g. vertical on M1, horizontal on M2). Pin access itself works similarly — a routing track has to actually cross a pin shape before TritonRoute will treat it as reachable, whether that pin sits on the same layer as the guide or one layer below.

Once guides are clean, TritonRoute works layer by layer using a panel-based scheme: each metal layer is split into panels, and it routes all panels on one layer in parallel before moving sequentially to the next layer up — intra-layer parallel, inter-layer sequential.

## SPEF and Post-Route STA

Once routing is done, the real resistance and capacitance of every wire can finally be extracted — these parasitics get written out to a SPEF (Standard Parasitic Exchange Format) file. That SPEF is back-annotated onto the netlist, and STA is re-run using the actual wire delays instead of estimates, giving the final sign-off timing numbers.

---

## Lab — Power Distribution, Routing

### Power Planning

Before routing, the power distribution network gets its own structure: a ring of VDD and GND running around the block boundary (the block power ring), with power stripes branching inward across the core to keep IR drop under control. Every standard cell row taps into this network through horizontal rail connections, so cell power pins never have to route far to reach a stripe. Macros like RAM blocks get their own local ring plus a halo — a keep-out margin around the macro that routing and placement both respect — and I/O pads sit on the outer padframe with their own power connections feeding in from the ring.

Generated the power distribution network:
```tcl
gen_pdn
```

Ran detailed routing:
```tcl
run_routing
```

Full routed die view after PDN generation and routing:

![Full routed chip - PDN and routing overview](images/49_day5.png)

Zoomed view of top-level I/O pins (`pcpi_rd`, `irq`, `trap`, `pcpi_cs`, `mem_rdata`, `mem_la_wdata`) landing at the chip boundary, with standard cell rows and filler cells visible below:

![Top-level I/O pin placement with standard cell rows](images/51_day5.png)

Zoomed view of a standard cell row showing fillers, decap cells, and antenna diode cells (labeled `ANTENNA`) threaded in among the logic cells:

![Standard cell row - fillers, decaps, and antenna cells](images/50_day5.png)

### Common Violations to Watch For

- **Min spacing violations** — two wires sitting too close together on the same layer
- **Antenna violations** — a long metal segment can accumulate charge during the etch step before the transistor above it is even connected, and that charge can punch through and damage the gate oxide
  - **Fix:** insert antenna diodes to bleed off the charge, or route a jumper via up to a higher layer to break the exposed segment

---

## Tools & Environment

| Tool | Purpose |
|---|---|
| OpenLANE | RTL-to-GDSII automation flow |
| Yosys | RTL synthesis |
| OpenROAD | Floorplan, placement, CTS, routing |
| Magic | Layout editor, DRC, LVS |
| OpenSTA | Static timing analysis |
| ngspice | SPICE simulation |
| TritonRoute | Detailed routing |
| Netgen | LVS (layout vs. schematic) |
| Sky130 PDK | SkyWater 130nm open-source PDK |

## Key Learnings

- Watched a design actually move through every stage from RTL to a manufacturable GDSII file, entirely on open-source tooling
- Built practical familiarity with floorplanning, placement, CTS, and routing by running them directly on the picorv32a core
- Learned the full path for bringing a hand-built standard cell into a flow that otherwise only knows the stock library
- Applied STA ideas that were abstract on paper — setup/hold slack, OCV, CRPR — by pulling real numbers out of OpenSTA
- Compared pre-route and post-route timing side by side and saw firsthand how much extracted parasitics can shift a design's numbers

## Acknowledgements

This workshop was put together by Kunal Ghosh (Co-founder, VSD Corp. Pvt. Ltd.) and Nickson P Jose (Physical Design Engineer, Intel), and I'm genuinely grateful for how deliberately it was structured — every concept came paired with a lab that made it concrete. Getting an actual RISC-V core through the entire RTL-to-GDSII flow using only open-source tools felt like a stretch goal going in; coming out the other side with a routed, sign-off-timed chip was a good reminder of how far this toolchain has come.

Credit where it's due: Kunal Ghosh for running VSD and designing this program, Nickson Jose for both his mentorship and the vsdstdcelldesign repo the Day 3 labs were built on, and NASSCOM for organizing and hosting the workshop.

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/digital-vlsi-soc-design-and-planning/) — course home page
- [OpenLANE](https://github.com/The-OpenROAD-Project/OpenLane) — the RTL-to-GDSII flow used throughout
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk) — the open-source PDK this whole build targets
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign) — Nickson Jose's repo for the custom cell lab

---
*Part of the [SoC Design of the PicoRV32 RISC-V micro-processor - VSD](https://github.com/saisindhumanne34/SoC-Design-of-the-PicoRV32-RISCV-micro-processor-VSD) workshop series.*
