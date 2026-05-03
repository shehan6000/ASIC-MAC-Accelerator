# ASIC MAC Accelerator

An end-to-end digital ASIC portfolio project: a signed fixed-point
multiply-accumulate accelerator written in SystemVerilog, verified with
simulation and formal checks, synthesized with Yosys, and taken through an
OpenLane/OpenROAD physical-design flow on SKY130.







## What This Project Demonstrates

This project is meant to show practical ASIC design skill, not just isolated RTL.
It covers:

- RTL microarchitecture and clean module partitioning
- APB-like memory-mapped register interface
- signed fixed-point datapath design
- control FSM design
- self-checking simulation
- formal safety checks
- synthesis with Yosys
- physical implementation with OpenLane/OpenROAD
- generated layout/GDS output
- engineering documentation and reproducible commands

## Design Overview

The top-level module is `mac_accelerator_top`.

The accelerator computes a signed dot product:

```text
result = sum(vector_a[i] * vector_b[i]), i = 0 .. length - 1
```

Default RTL configuration:

| Item | Value |
| --- | --- |
| Input element width | signed 16-bit |
| Accumulator width | signed 40-bit |
| Result width | signed 32-bit saturated |
| Max vector length | 256 elements |
| Bus interface | APB-like |
| Interrupt | `irq` asserted when operation is done and enabled |

Physical OpenLane build:

| Item | Value |
| --- | --- |
| PDK | SKY130A |
| Standard cell library | `sky130_fd_sc_hd` |
| Physical build vector depth | `MAX_LEN=32` |
| Clock target | 50 ns / 20 MHz |
| Post-PNR estimated min period | 15.02 ns |
| Estimated Fmax | 66.57 MHz |
| Setup violations at nominal TT | 0 |
| Hold violations at nominal TT | 0 |

The RTL supports the larger 256-entry buffer. The OpenLane configuration uses
`MAX_LEN=32` so the inferred register-array buffer can complete physical design
without SRAM macros. A production version should replace the operand buffers
with SRAM macros.

## Block Diagram

```text
                 APB-like Host Bus
                         |
                         v
              +----------------------+
              |  Register Interface  |
              | CTRL STATUS LENGTH   |
              | MODE RESULT IRQ_EN   |
              +----------+-----------+
                         |
                         v
              +----------------------+
              |     Control FSM      |
              |   IDLE / RUN / DONE  |
              +----------+-----------+
                         |
          +--------------+--------------+
          |                             |
          v                             v
  +---------------+             +---------------+
  | Vector A RAM  |             | Vector B RAM  |
  | register impl |             | register impl |
  +-------+-------+             +-------+-------+
          |                             |
          +--------------+--------------+
                         |
                         v
              +----------------------+
              |  Signed MAC Datapath |
              |  16x16 -> 40-bit acc |
              +----------+-----------+
                         |
                         v
              +----------------------+
              | Saturation / Result  |
              +----------------------+
```

## Repository Layout

```text
rtl/                 Synthesizable SystemVerilog RTL
tb/iverilog/         Self-checking Icarus Verilog testbench
tb/cocotb/           cocotb testbench and Python golden-model tests
tb/formal/           SymbiYosys harness and properties
sim/                 RTL file list
synth/               Yosys synthesis script
openlane/            OpenLane SKY130 configuration
scripts/             Windows/OpenLane helper scripts
docs/                Architecture, verification, and physical-design notes
images/              Layout preview image
reports/             Generated synthesis reports
runs/                Generated OpenLane run output, not required in Git
```

## Register Map

| Address | Name | Description |
| --- | --- | --- |
| `0x000` | `CTRL` | bit 0 `start`, bit 1 `clear`, bit 2 `irq_enable` |
| `0x004` | `STATUS` | bit 0 `busy`, bit 1 `done`, bit 2 `overflow`, bit 3 `irq` |
| `0x008` | `LENGTH` | vector length, 1 to 256 |
| `0x00C` | `MODE` | reserved for future modes, mode 0 is dot product |
| `0x010` | `RESULT_LO` | signed 32-bit saturated result |
| `0x014` | `RESULT_HI` | sign extension of `RESULT_LO` |
| `0x100`-`0x1FF` | `VECTOR_A` | signed 16-bit vector A elements |
| `0x200`-`0x2FF` | `VECTOR_B` | signed 16-bit vector B elements |

## Quick Start On Windows

This repository includes helper scripts for a local OSS CAD Suite installation.

```powershell
cd asic-mac-accelerator
.\scripts\run_windows_tools.ps1 versions
.\scripts\run_windows_tools.ps1 lint
.\scripts\run_windows_tools.ps1 sim
.\scripts\run_windows_tools.ps1 formal
.\scripts\run_windows_tools.ps1 synth
```

Expected simulation output:

```text
PASS: result 70
PASS: result -21
SIM PASS
```

The simulation waveform is generated at:

```text
sim_build/tb_mac_accelerator.vcd
```

Open it with GTKWave:

```powershell
gtkwave sim_build\tb_mac_accelerator.vcd
```

Useful waveform signals:

```text
tb_mac_accelerator.dut.busy
tb_mac_accelerator.dut.done
tb_mac_accelerator.dut.irq
tb_mac_accelerator.dut.rd_index
tb_mac_accelerator.dut.result_sat
tb_mac_accelerator.dut.u_control_fsm.acc_q
tb_mac_accelerator.dut.u_control_fsm.state_q
```

## Linux / WSL Commands

With Yosys, Icarus Verilog, Verilator, SymbiYosys, and cocotb installed:

```bash
make lint
make sim
make formal
make synth
```

## Physical Design With OpenLane

The project includes a Docker helper for OpenLane 2:

```powershell
.\scripts\run_openlane_docker.ps1 -SmokeTest
.\scripts\run_openlane_docker.ps1 -RunTag mac_accelerator
```

Generated layout files:

```text
runs/mac_accelerator/58-klayout-streamout/mac_accelerator_top.klayout.gds
runs/mac_accelerator/57-magic-streamout/mac_accelerator_top.gds
runs/mac_accelerator/59-magic-writelef/mac_accelerator_top.lef
```

Render a preview from the GDS:

```powershell
docker run --rm -v "${PWD}:/work" -w /work ghcr.io/efabless/openlane2:2.3.10 klayout -b -r scripts/render_gds.py
```

## Verification

The project includes three verification layers:

| Layer | Tool | Purpose |
| --- | --- | --- |
| RTL lint | Verilator | catches structural RTL issues |
| Simulation | Icarus Verilog | self-checking directed tests |
| Formal | SymbiYosys | bounded safety checks |

The self-checking simulation verifies:

- positive dot product
- mixed signed operands
- APB register writes
- start/done sequencing
- result readback

The formal harness checks safety properties such as:

- `busy` and `done` are not asserted together
- `irq` implies `done`
- run index stays within configured length
- bus ready behavior remains stable

## Current Results

| Check | Status |
| --- | --- |
| Verilator lint | Pass |
| Icarus compile | Pass |
| Self-checking simulation | Pass |
| SymbiYosys BMC | Pass |
| Yosys synthesis | Pass |
| OpenLane GDS streamout | Generated |
| KLayout XOR | Clear |
| Post-PNR nominal setup violations | 0 |
| Post-PNR nominal hold violations | 0 |

Note: the OpenLane run produced GDS layout output. The later signoff tail can be
long on Windows/Docker and may need to be resumed or rerun for fully packaged
final reports.
