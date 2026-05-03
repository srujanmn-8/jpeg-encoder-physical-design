# JPEG Encoder ASIC — RTL-to-GDSII Physical Design

![Nangate 45nm](https://img.shields.io/badge/Process-Nangate%2045nm-blue)
![OpenROAD](https://img.shields.io/badge/Tool-OpenROAD-green)
![Timing](https://img.shields.io/badge/Timing-All%206%20Constraints%20MET-brightgreen)
![DRC](https://img.shields.io/badge/DRC-0%20Errors-brightgreen)
![LVS](https://img.shields.io/badge/LVS-PASS-brightgreen)

## Overview

This repository documents the complete **RTL-to-GDSII physical design implementation** of a JPEG Encoder IP using the OpenROAD open-source EDA flow on the **Nangate 45nm (FreePDK)** standard cell library.

The design was swept across **six clock frequency constraints** — from 25 MHz to 1 GHz — to study the effect of clock period on PPA (Power, Performance, Area) metrics. Timing closure was achieved at all six operating points.

> **Note on RTL source:** The RTL Verilog design used as input is the reference JPEG Encoder from the OpenROAD-flow-scripts repository:  
> https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts/tree/master/flow/designs/src/jpeg  
> Licensed under Apache 2.0. This project focuses entirely on the **physical design implementation, analysis, and optimization** of that design.

---

## Architecture

The JPEG Encoder implements a 3-stage synchronous pipelined datapath:

```
Input pixels (8-bit)
        │
        ▼
┌───────────────┐
│  FDCT Stage   │  Forward Discrete Cosine Transform
│  (fdct.v)     │  + ZigZag reordering
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  QNR Stage    │  Quantization and Rounding
│ (jpeg_qnr.v) │  (div_su.v, div_uu.v)
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  RLE Stage    │  Run-Length Encoding
│ (jpeg_rle.v) │  (jpeg_rle1.v, jpeg_rzs.v)
└───────┬───────┘
        │
        ▼
Encoded output (size, rlen, amp, douten)
```

**Top module:** `jpeg_encoder.v`  
**Total RTL files:** 13 hierarchical Verilog modules  
**Reset:** Asynchronous active-low  
**Inputs:** `clk`, `rst`, `ena`, `dstrb`, `qnt_val[6:0]`, `din[7:0]`  
**Outputs:** `size[3:0]`, `rlen[3:0]`, `amp[10:0]`, `douten`

---

## Physical Design Flow

```
Verilog RTL (13 modules) + SDC constraints
               │
               ▼
     ┌──────────────────┐
     │    Synthesis      │  Yosys + ABC → gate-level netlist
     │  (Nangate45 lib)  │  mapped to standard cells
     └────────┬─────────┘
              │  .lef + netlist
              ▼
     ┌──────────────────┐
     │  Floorplanning   │  Die area ~90,000–95,000 µm²
     │   (OpenROAD)     │  
     └────────┬─────────┘
              │
              ▼
     ┌──────────────────┐
     │   Placement      │  RePlAce (global) + OpenDP (detailed)
     │                  │  Density target: 20%
     └────────┬─────────┘
              │
              ▼
     ┌──────────────────┐
     │  Clock Tree      │  TritonCTS → balanced H-tree
     │  Synthesis (CTS) │  Minimal skew across all flip-flops
     └────────┬─────────┘
              │
              ▼
     ┌──────────────────┐
     │    Routing       │  FastRoute (global) + TritonRoute (detailed)
     │                  │  10 metal layers (M1–M10), 520,601 vias
     └────────┬─────────┘
              │
              ▼
     ┌──────────────────┐
     │  Verification    │  DRC: 0 errors | LVS: PASS
     │  + GDS Export    │  KLayout → final GDSII file
     └──────────────────┘
```

---

## Results

### Frequency Sweep — Full PPA Table

| # | Frequency | Clock Period | Total Power | Internal Power | Switching Power | Leakage Power | Setup Slack | Timing | Core Area | Utilization | IR Drop VDD | IR Drop VSS | Total Vias |
|---|-----------|-------------|------------|---------------|----------------|--------------|------------|--------|-----------|-------------|-------------|-------------|------------|
| 1 | 1 GHz     | 1 ns        | 592.0 mW   | 318.0 mW      | 272.0 mW        | 2.10 mW      | 0.55 ns    | ✅ MET | 90,729 µm² | 20%        | 0.80%       | 1.31%       | 520,601    |
| 2 | 125 MHz   | 8 ns        | 70.9 mW    | 35.5 mW       | 33.5 mW         | 1.93 mW      | 6.43 ns    | ✅ MET | 90,060 µm² | 20%        | 0.51%       | 0.41%       | 520,601    |
| 3 | 100 MHz   | 10 ns       | 57.1 mW    | 28.4 mW       | 26.8 mW         | 1.93 mW      | 8.11 ns    | ✅ MET | 90,072 µm² | 20%        | 0.52%       | 0.42%       | 520,601    |
| 4 | 75 MHz    | 13.33 ns    | 43.4 mW    | 21.3 mW       | 20.1 mW         | 1.93 mW      | 10.76 ns   | ✅ MET | 90,084 µm² | 20%        | 0.49%       | 0.39%       | 520,601    |
| 5 | 50 MHz    | 20 ns       | 29.5 mW    | 14.2 mW       | 13.4 mW         | 1.93 mW      | 16.02 ns   | ✅ MET | 90,075 µm² | 20%        | 0.55%       | 0.42%       | 520,601    |
| 6 | 25 MHz    | 40 ns       | 15.7 mW    | 7.09 mW       | 6.70 mW         | 1.93 mW      | 31.97 ns   | ✅ MET | 90,046 µm² | 20%        | 0.44%       | 0.43%       | 520,601    |

### Summary Statistics

| Parameter        | Min        | Max        | Average    |
|-----------------|------------|------------|------------|
| Total Power      | 15.7 mW    | 592.0 mW   | 135.0 mW   |
| Internal Power   | 7.09 mW    | 318.0 mW   | 70.7 mW    |
| Switching Power  | 6.70 mW    | 272.0 mW   | 62.1 mW    |
| Leakage Power    | 1.93 mW    | 2.10 mW    | 1.96 mW    |
| IR Drop VDD      | 1.15 mV    | 3.07 mV    | 1.48 mV    |
| IR Drop VSS      | 1.16 mV    | 3.18 mV    | 1.50 mV    |
| Setup Slack      | 0.55 ns    | 31.97 ns   | 12.31 ns   |
| Core Area        | 90,046 µm² | 94,729 µm² | 90,844 µm² |

### Physical Signoff

| Check               | Result         |
|--------------------|----------------|
| DRC Errors          | **0** ✅       |
| Antenna Violations  | **0** ✅       |
| LVS Check           | **PASS** ✅    |
| Max IR Drop (VDD)   | 0.80% @ 1 GHz ✅ |
| Max IR Drop (VSS)   | 1.31% @ 1 GHz ✅ |
| Total Vias          | 520,601        |

---

### 6. Recommended operating points

| Use Case                      | Frequency | Power   | Reason                                      |
|------------------------------|-----------|---------|---------------------------------------------|
| ⭐ Nominal / production        | 125 MHz   | 70.9 mW | 6.43 ns slack, 20% utilization, balanced PPA |
| Ultra-low-power IoT           | 25 MHz    | 15.7 mW | Minimum power, leakage contributes 12.3%    |
| Maximum throughput            | 1 GHz     | 592 mW  | Marginal slack, use only if speed critical   |

---

## Tools Used

| Tool | Version / Source | Purpose |
|------|-----------------|---------|
| Verilog HDL | — | RTL description |
| Yosys + ABC | OpenROAD-flow-scripts | Logic synthesis |
| OpenROAD | open-source | floorplan, place, CTS, route |
| RePlAce | OpenROAD built-in | Global placement (ePlace algorithm) |
| OpenDP | OpenROAD built-in | Detailed placement legalization |
| TritonCTS | OpenROAD built-in | Clock tree synthesis |
| FastRoute | OpenROAD built-in | Global routing |
| TritonRoute | OpenROAD built-in | Detailed routing |
| KLayout | v0.28+ | GDSII generation, DRC, LVS |
| Nangate 45nm FreePDK | SI2 / FreePDK | Standard cell library |

---

## References

1. OpenROAD Project — https://github.com/The-OpenROAD-Project/OpenROAD
2. OpenROAD-flow-scripts JPEG design (RTL source) — https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts/tree/master/flow/designs/src/jpeg
3. Nangate 45nm FreePDK — https://si2.org/openeda.si2.org/projects/nangatelib
4. Yosys Open Synthesis Suite — https://yosyshq.net/yosys
5. KLayout GDS viewer and DRC — https://www.klayout.de
6. P. Bhatt and S. Jadhav, "Comparative Analysis of Physical Design Parameters for JPEG Encoder IP using OpenROAD," IEEE AIIoT 2024, doi: 10.1109/AIIoT61789.2024.10579002

---
