# SoC Design of PicoRV32 RISC-V Micro-Processor

End-to-end Physical Design flow from RTL to GDSII for the PicoRV32 RISC-V processor using OpenLANE and Sky130 PDK.

**Workshop:** VSD SoC Design and Planning  
**Tools:** OpenLANE | Magic | OpenSTA | NGSpice | Sky130 PDK  
**Design:** PicoRV32 RISC-V Core  

---

## Day 1 — Introduction to Open-Source EDA and OpenLANE Flow

### What I Learned
Every ASIC design requires three key components working together:
- **RTL Design** — describes what the chip should do in hardware description language
- **EDA Tools** — software that converts RTL into a physical chip layout
- **PDK Data** — manufacturing rules and cell libraries provided by the foundry

The **Sky130 PDK** released by Google and SkyWater Technology made chip design open source for the first time — a major milestone for the VLSI community.

**OpenLANE** is a fully automated RTL to GDSII flow that uses multiple open-source tools to take a design from RTL netlist all the way to a manufacturable layout file.

### RTL to GDSII Flow Overview
RTL → Synthesis → Floorplan → Placement → CTS → Routing → Sign-off → GDSII
