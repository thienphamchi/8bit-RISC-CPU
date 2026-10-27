# 8-bit RISC Processor Core with a 5-Stage Pipeline

A small, fully synthesizable 8-bit RISC CPU written in Verilog-2001, with a classic
5-stage pipeline (IF, ID, EX, MEM, WB), data forwarding, load-use stall and branch
flush. It comes with a self-checking testbench and a Vivado-ready file set.

- **Language:** Verilog-2001
- **Tool:** Vivado (project created in Vivado 2026.1)
- **Target part:** `xa7s100fgga676-1Q` (XA Spartan-7); the code is device-independent
- **Clock target:** 100 MHz (10 ns)

---

## 1. Package contents

```
risc8_vivado/
├── design_sources/            add as Design Sources (7 files)
│   ├── risc8_5stage_core.v    top level: PC, 4 pipeline registers, flags, branch/halt logic
│   ├── imem.v                 instruction ROM 256 x 16, program built in (case statement)
│   ├── control_unit.v         opcode -> control signals (combinational)
│   ├── regfile.v              8 x 8-bit registers, 2 read ports, 1 write port, write-through bypass
│   ├── hazard_unit.v          forwarding select + load-use stall (combinational)
│   ├── alu.v                  8 operations, produces Z and C (combinational)
│   └── dmem.v                 data RAM 256 x 8 (LD / ST only)
├── sim_sources/               add as Simulation Sources
│   └── tb_risc8_5stage.v      self-checking testbench with a 5-stage text trace
├── constraints/
│   └── risc8.xdc              100 MHz clock constraint + pin template (pins to be filled in)
└── README.md
```

> Comments inside the `.v` files still mention `rtl/` and `tb/`. That is the folder
> naming of the original package; it does not affect Vivado.

---

## 2. Specification

| Parameter | Value |
|---|---|
| Data width | 8 bits |
| Instruction width | 16 bits, fixed |
| Registers | 8 x 8-bit (R0-R7). R0 is an ordinary register, not hard-wired to zero |
| Program counter | 8 bits (256 instructions) |
| Instruction memory | 256 x 16 ROM (asynchronous read) |
| Data memory | 256 x 8 RAM (asynchronous read, synchronous write) |
| Architecture | Harvard (separate instruction and data memories) |
| Pipeline | 5 stages: IF, ID, EX, MEM, WB |
| Flags | Z (zero), C (carry / borrow) |
| Instructions | 16 (4-bit opcode) |
| Hazard handling | Forwarding + register-file bypass, 1-cycle load-use stall, 2-cycle branch flush |
| Reset | Synchronous, active high |

### Ports of `risc8_5stage_core`

| Port | Dir | Description |
|---|---|---|
| `clk` | in | System clock |
| `rst` | in | Synchronous reset, active high |
| `halted` | out | 1 after HLT has left the pipeline |
| `pc_out[7:0]` | out | Current PC (debug) |
| `flags_out[1:0]` | out | `{Z, C}` (debug) |
| `instr_done` | out | Pulses when an instruction completes WB (debug) |

The program is inside `imem.v`, so the core has only `clk` and `rst` as inputs.

---

## 3. Instruction set architecture

### Formats (bit 15 on the left)

```
R-type  | opcode(4) | rd(3) | rs1(3) | rs2(3) | 000(3)        ADD SUB AND OR XOR
R1-type | opcode(4) | rd(3) | rs1(3) | 000000(6)              SHL SHR LD ST
I-type  | opcode(4) | rd(3) | 0 | imm8(8)                     LDI ADDI
J-type  | opcode(4) | 0000 | addr8(8)                         JMP BZ BNZ
```

HLT and NOP use only the opcode; all other bits are 0.

### Instructions

| Opcode | Mnemonic | Operation | Flags |
|---|---|---|---|
| 0 | NOP | do nothing | - |
| 1 | ADD rd,rs1,rs2 | rd = rs1 + rs2 | Z, C |
| 2 | SUB rd,rs1,rs2 | rd = rs1 - rs2 | Z, C (borrow when rs1 < rs2) |
| 3 | AND rd,rs1,rs2 | rd = rs1 & rs2 | Z, C = 0 |
| 4 | OR rd,rs1,rs2 | rd = rs1 \| rs2 | Z, C = 0 |
| 5 | XOR rd,rs1,rs2 | rd = rs1 ^ rs2 | Z, C = 0 |
| 6 | SHL rd,rs1 | rd = rs1 << 1 | Z, C = old bit 7 |
| 7 | SHR rd,rs1 | rd = rs1 >> 1 | Z, C = old bit 0 |
| 8 | LDI rd,imm8 | rd = imm8 | - |
| 9 | ADDI rd,imm8 | rd = rd + imm8 | Z, C |
| A | LD rd,[rs1] | rd = mem[rs1] | - |
| B | ST rd,[rs1] | mem[rs1] = rd | - |
| C | JMP addr8 | PC = addr8 | - |
| D | BZ addr8 | if Z = 1 then PC = addr8 | - |
| E | BNZ addr8 | if Z = 0 then PC = addr8 | - |
| F | HLT | stop the processor | - |

Encoding examples: `ADD R2,R2,R1` = `0001 010 010 001 000` = `0x1488`;
`LDI R1,0x3C` = `1000 001 0 0011 1100` = `0x823C`.

> Note: the original PDF guide lists the logic instructions as updating only Z. In the
> RTL, every instruction with `flag_we` writes both flags, so logic instructions clear C.
> This README follows the RTL.

---

## 4. Microarchitecture

### Modules

| File | Module | Stage | Kind |
|---|---|---|---|
| `risc8_5stage_core.v` | `risc8_5stage_core` | all | sequential + wiring |
| `imem.v` | `imem` | IF | ROM (combinational case) |
| `control_unit.v` | `control_unit` | ID | combinational |
| `regfile.v` | `regfile` | ID / WB | sequential |
| `hazard_unit.v` | `hazard_unit` | ID / EX | combinational |
| `alu.v` | `alu` | EX | combinational |
| `dmem.v` | `dmem` | MEM | RAM |

### Pipeline registers

| Register | Width | Contents |
|---|---|---|
| IF/ID | 17 | instruction (16), valid |
| ID/EX | 48 | valid, 14 control bits, rd, 2 register numbers, 2 register values, imm8 |
| EX/MEM | 32 | valid, reg_we, mem_to_reg, mem_we, halt, rd, ALU result, address, store data |
| MEM/WB | 14 | valid, reg_we, halt, rd, result |

A pipeline slot with `valid = 0` is a **bubble**: all its write-enables are forced to 0,
so it cannot change registers, memory or flags.

### Hazards

| Kind | Solution | Cost |
|---|---|---|
| Structural | Harvard memories; register file with separate read/write ports | 0 |
| Data (RAW) | Forwarding from EX/MEM and MEM/WB; write-through bypass in regfile | 0 cycles |
| Load-use | 1-cycle stall, then forwarding from MEM/WB | 1 cycle |
| Control | Branch resolved in EX; the 2 younger instructions are flushed | 2 cycles per taken branch |

Forwarding select codes (per operand, in `hazard_unit.v`):
`2'b00` = value read in ID, `2'b10` = EX/MEM (newest wins), `2'b01` = MEM/WB.
Forwarding is gated by `uses_a` / `uses_b`, so unused register fields never cause a false match.

**HLT:** when HLT reaches EX, fetching stops and younger instructions are squashed; older
instructions finish through MEM and WB, then `halted` becomes 1 and the pipeline freezes.

---

## 5. Built-in test program

`imem.v` contains a program that covers every instruction, dependencies at distance 1-3,
a load-use pair, taken and not-taken branches, wrong-path instructions that must be
cancelled, a loop (5+4+3+2+1), a carry, a jump and HLT.

| Addr | Hex | Instruction | Comment |
|---|---|---|---|
| 00 | 823C | LDI R1,0x3C | |
| 01 | 840F | LDI R2,0x0F | |
| 02 | 3650 | AND R3,R1,R2 | R3 = 0x0C |
| 03 | 4850 | OR R4,R1,R2 | R4 = 0x3F |
| 04 | 5AE0 | XOR R5,R3,R4 | R5 = 0x33 |
| 05 | 6D40 | SHL R6,R5 | R6 = 0x66 |
| 06 | 7F80 | SHR R7,R6 | R7 = 0x33 |
| 07 | 2FE8 | SUB R7,R7,R5 | R7 = 0, Z = 1 |
| 08 | D00A | BZ 0x0A | taken |
| 09 | 8EFF | LDI R7,0xFF | wrong path, flushed |
| 0A | 8205 | LDI R1,5 | loop counter |
| 0B | 8400 | LDI R2,0 | sum |
| 0C | 8601 | LDI R3,1 | constant 1 |
| 0D | 1488 | ADD R2,R2,R1 | loop start |
| 0E | 2258 | SUB R1,R1,R3 | sets Z at 0 |
| 0F | E00D | BNZ 0x0D | taken 4 times |
| 10 | 8820 | LDI R4,0x20 | address 0x20 |
| 11 | B500 | ST R2,[R4] | MEM[0x20] = 15 |
| 12 | AB00 | LD R5,[R4] | R5 = 15 |
| 13 | 9AF0 | ADDI R5,0xF0 | R5 = 0xFF (load-use) |
| 14 | 9A01 | ADDI R5,1 | R5 = 0, C = 1, Z = 1 |
| 15 | C017 | JMP 0x17 | |
| 16 | 8CEE | LDI R6,0xEE | wrong path, flushed |
| 17 | F000 | HLT | |
| 18 | 8CAA | LDI R6,0xAA | never executed |

Expected final state: `R1..R7 = 00 0F 01 20 00 66 00`, `MEM[0x20] = 0F`, `Z = 1, C = 1`.

### Changing the program

Edit the lines in the `case (addr)` block of `imem.v`, one line per address:

```verilog
8'h00 : instr = 16'h823C;   // LDI R1, 0x3C
```

Addresses that are not listed return `0x0000` (NOP). Then update the expected values in
the check section of `tb_risc8_5stage.v`.

---

## 6. Quick start in Vivado

### Create the project

1. **Create Project** -> RTL Project. Use a path without spaces.
2. **Add Sources** -> Add Files -> the 7 files in `design_sources/`. Tick *Copy sources into project*.
3. **Add Constraints** -> `constraints/risc8.xdc`.
4. **Default Part:** `xa7s100fgga676-1Q`.
5. In *Design Sources*, right-click `risc8_5stage_core` -> **Set as Top**.
   (Vivado may pick `alu` automatically. Check *Top module name* in the Project Summary.)
6. **File -> Add Sources -> Add or create simulation sources** -> `sim_sources/tb_risc8_5stage.v`.
   In *Simulation Sources -> sim_1*, right-click `tb_risc8_5stage` -> **Set as Top**.

### Behavioral simulation

**Run Simulation -> Run Behavioral Simulation.** The test stops by itself at **545 ns**.
In the Tcl Console you should see:

```
PASS  R1 = 00
PASS  R2 = 0f
PASS  R3 = 01
PASS  R4 = 20
PASS  R5 = 00
PASS  R6 = 66
PASS  R7 = 00
PASS  MEM[20] = 0F
PASS  flags Z=1 C=1

cycles=51  instructions=34  CPI=1.50  load-use stalls=1  taken branches=6  forwards=12
*** ALL TESTS PASSED ***
```

If you see `TIMEOUT` or `FAIL`, read the per-cycle trace above the summary (next section).
The testbench also writes `risc8_5stage.vcd`; it is a generated waveform file and can be deleted.

### Synthesis, implementation, timing

1. **Run Synthesis.** Check Messages/Log: there must be no `latch` warning, and
   *Report Utilization* should show `Register as Latch = 0`.
   Optional check in the Tcl Console of the synthesized design:
   `get_cells -hier -filter {REF_NAME =~ LD*}` should match nothing.
2. **Run Implementation**, then **Report Timing Summary**. WNS and WHS must be >= 0.
3. **Generate Bitstream** needs real pin assignments (see section 8).

### Do not use

Post-synthesis / post-implementation *functional* simulation with this testbench: it reads
internal signals by hierarchical name (`dut.stall`, `dut.u_rf.r[...]`, ...) that do not
survive in the netlist. Use Behavioral Simulation.

---

## 7. The testbench and its trace

`tb_risc8_5stage.v` generates a 100 MHz clock, holds reset for about two cycles, runs the
program until `halted`, checks the result and ends. A watchdog (`TIMEOUT` at 20 us)
prevents a hang.

### Per-cycle trace

Each clock cycle prints one line with the instruction in **every** stage, as text:

```
t=125000 PC=0a | IF:(flushed) | ID:LDI  R7,0xFF | EX:BZ   0x0A | MEM:SUB  R7,R7,R5 | WB:... | Z=1 C=1 BRANCH->flush 2
```

- Empty slots show `(bubble)`; IF shows `(flushed)` when its instruction will be discarded.
- Markers at the end of the line: `STALL`, `BRANCH->flush 2`, `fwd`.
- The WB column also shows the register write, for example `R3<=0C`.

### How it works

- **Mini disassembler:** Verilog functions (`dis`, `hexch`, `regch`) turn each 16-bit word
  into fixed-width text, so the columns always line up.
- **Shadow registers** `sh_ex`, `sh_mem`, `sh_wb`: the core carries only control bits down
  the pipeline, not the instruction, so the testbench follows the instruction with its own
  copies, using the same move / hold / bubble rules as the core.
- The testbench only **reads** core signals. The RTL is not modified, so the synthesized
  design is exactly the verified design.

**Limit:** the shadow registers copy the core's stall/flush rules. If those rules change in
the core, update the testbench as well.

---

## 8. Constraints and board pins

`risc8.xdc` has two parts:

1. **Done:** `create_clock -period 10.000` (100 MHz), `CFGBVS VCCO`, `CONFIG_VOLTAGE 3.3`.
   This is enough for synthesis, implementation and the timing report.
   Check `CONFIG_VOLTAGE` against the VCCO of the configuration bank on your board
   (3.3 V or 1.8 V), and set the real period if your oscillator is not 100 MHz.
2. **To fill in:** a commented template for `clk`, `rst`, `halted`, `instr_done`,
   `flags_out[1:0]` and `pc_out[7:0]`. Replace `<pin>` with the ball name from your board
   schematic and `<std>` with the IO standard of that bank, then remove the `##`.
   Without this, **Generate Bitstream will fail**.

`rst` is active high. If your button is active low, invert it before the core.
On a real board the program finishes in about half a microsecond, so to watch it on LEDs
put a clock divider (for example to 1 Hz) between the board clock and `clk`.

---

## 9. Results

### Simulation (verified)

| Item | Result |
|---|---|
| Directed test | 9/9 checks PASS |
| Cycles / instructions / CPI | 51 / 34 / 1.50 (4 fill + 12 flush + 1 stall cycles) |
| Load-use stalls / taken branches / forwarded operands | 1 / 6 / 12 |
| Run time | stops at 545 ns |

### Vivado synthesis and timing (author's run, `xa7s100fgga676-1Q`)

| Item | Result |
|---|---|
| Slice LUTs | 237 (0.37 %): 205 as logic, 32 as distributed RAM |
| Slice registers | 187 flip-flops (0.15 %) |
| Latches | 0 |
| F7 / F8 muxes | 24 / 8 |
| WNS (setup), 100 MHz | +2.236 ns |
| WHS (hold) | +0.040 ns |
| Fmax estimate | about 129 MHz = 1 / (10 - 2.236) ns |

The 32 LUT-RAM cells plus F7/F8 muxes are the 256 x 8 data memory.
WHS is thin; re-check WNS/WHS in the **Implementation** report (real routing) if these
numbers were taken after synthesis only.

### Extra checks done outside Vivado

| Check | Result |
|---|---|
| Verilator `-Wall` lint (RTL and testbench) | no warnings |
| Icarus Verilog `-Wall` compile | no warnings |
| Random programs vs. a Python reference model (instruction-set simulator) | 2500 programs, 0 mismatches (500 from the original package + 2000 extra with another seed) |

The random test belongs to the original Icarus/Python package (same pipeline RTL, program
loaded from a `.hex` file). The Python tools are **not** part of this Vivado package.

---

## 10. Status and limitations

- Verified: behavioral simulation, lint, random regression (original package), synthesis
  without latches, timing report at 100 MHz.
- Not done yet: pin assignment, bitstream and a test on the physical board.
- `imem` and `dmem` use asynchronous reads (distributed ROM/RAM). Synchronous block RAM
  would need one more cycle of latency in IF and MEM.
- Branches resolve in EX, so a taken branch costs 2 cycles.
- The testbench is written for the built-in program; a new program needs new expected values.
