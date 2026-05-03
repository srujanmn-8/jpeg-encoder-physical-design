ASIC RTL-to-GDSII physical design of a JPEG Encoder IP using OpenROAD on Nangate 45nm

Overview
This repository documents the complete RTL-to-GDSII physical design implementation of a JPEG Encoder IP using the OpenROAD open-source EDA flow on the Nangate 45nm (FreePDK) standard cell library.
The design was swept across six clock frequency constraints — from 25 MHz to 1 GHz — to study the effect of clock period on PPA (Power, Performance, Area) metrics. Timing closure was achieved at all six operating points.
-----------------------------------------------------------------------------------------------------------------------
Note on RTL source: The RTL Verilog design used as input is the reference JPEG Encoder from the OpenROAD-flow-scripts repository:
https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts/tree/master/flow/designs/src/jpeg
Licensed under Apache 2.0. This project focuses entirely on the physical design implementation, analysis, and optimization of that design.
-----------------------------------------------------------------------------------------------------------------------
Architecture
The JPEG Encoder implements a 3-stage synchronous pipelined datapath:
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
Top module: jpeg_encoder.v
Total RTL files: 13 hierarchical Verilog modules
Reset: Asynchronous active-low
Inputs: clk, rst, ena, dstrb, qnt_val[6:0], din[7:0]
Outputs: size[3:0], rlen[3:0], amp[10:0], douten
-----------------------------------------------------------------------------------------------------------------------
Physical Design Flow
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

----------------------------------------------------------------------------------------------------------------------
Results
Frequency Sweep — Full PPA Table
| Frequency | Clock Period | Total Power | Setup Slack | Timing  | Core Area | Utilization |
|-----------|------------- |----------- -|--------- ---|---------|-----------|-------------|
| 1 GHz     | 1 ns         | 592.0 mW    | 0.55 ns     | ✅ MET | 90,729 µm² | 20%        |
| 125 MHz   | 8 ns         | 70.9 mW     | 6.43 ns     | ✅ MET | 90,060 µm² | 20%        |
| 100 MHz   | 10 ns        | 57.1 mW     | 8.11 ns     | ✅ MET | 90,072 µm² | 20%        |
| 75 MHz    | 13.33 ns     | 43.4 mW     | 10.76 ns    | ✅ MET | 90,084 µm² | 20%        |
| 50 MHz    | 20 ns        | 29.5 mW     | 16.02 ns    | ✅ MET | 90,075 µm² | 20%        |
| 25 MHz    | 40 ns        | 15.7 mW     | 31.97 ns    | ✅ MET | 90,046 µm² | 20%        |
-----------------------------------------------------------------------------------------------------------------------
Physical Signoff
| Check               | Result            |
|------------------- -|-------------------|
| DRC Errors          | 0 ✅              |
| Antenna Violations  | 0 ✅              |
| LVS Check           | PASS ✅           |
| Max IR Drop (VDD)   | 0.80% @ 1 GHz ✅  |
| Max IR Drop (VSS)   | 1.31% @ 1 GHz ✅  |
| Total Vias          | 520,601            |

-----------------------------------------------------------------------------------------------------------------------
Tools Used

| Tool                  | Purpose                                        |
|-----------------------|------------------------------------------------|
| Verilog HDL           | RTL design (13 hierarchical modules)           |
| Yosys + ABC           | Logic synthesis → gate-level netlist           |
| RePlAce               | Global placement (ePlace algorithm)            |
| OpenDP                | Detailed placement legalization                |
| TritonCTS             | Clock tree synthesis (balanced H-tree)         |
| FastRoute             | Global routing                                 |
| TritonRoute           | Detailed routing (M1–M10, 520,601 vias)        |
| KLayout               | GDSII generation + DRC/LVS signoff             |
| Nangate 45nm FreePDK  | Standard cell library                          |
| OpenSTA               | Static timing analysis (integrated in OpenROAD)|

-----------------------------------------------------------------------------------------------------------------------
References

• OpenROAD Project — https://github.com/The-OpenROAD-Project/OpenROAD
OpenROAD-flow-scripts JPEG design (RTL source) — https://github.com/The-OpenROAD-Project/OpenROAD-flow-scripts/tree/master/flow/designs/src/jpeg
• Nangate 45nm FreePDK — https://si2.org/openeda.si2.org/projects/nangatelib
• Yosys Open Synthesis Suite — https://yosyshq.net/yosys
• KLayout GDS viewer and DRC — https://www.klayout.de
• P. Bhatt and S. Jadhav, "Comparative Analysis of Physical Design Parameters for JPEG Encoder IP using OpenROAD," IEEE AIIoT 2024, doi: 10.1109/AIIoT61789.2024.10579002
