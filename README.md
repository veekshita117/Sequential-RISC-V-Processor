# Sequential RISC-V Processor (RV64I)

A 64-bit single-cycle RISC-V processor implemented in Verilog, supporting a core subset of the RV64I instruction set. Every instruction — fetch, decode, execute, memory access, and write-back — completes in a single clock cycle.

This project was built as part of the *Introduction to Processor Architecture* course at IIIT Hyderabad, and serves as the foundation for a future pipelined implementation.

## Features

- **64-bit datapath** with a 32-entry, 64-bit register file (`x0` hardwired to zero)
- **Single-cycle control** — one instruction executes per clock cycle
- **Structural ALU** built from gate-level full adders (ripple-carry add/subtract) plus bitwise AND/OR
- Byte-addressable **instruction memory** (4096 bytes) initialized from a hex program file
- Byte-addressable **data memory** (1024 bytes) for loads and stores
- Modular design — PC, control unit, register file, immediate generator, ALU control, ALU, and data memory are all separate, independently-tested modules

## Supported Instructions

| Type    | Instructions           | Notes                                  |
|---------|------------------------|-----------------------------------------|
| R-type  | `add`, `sub`, `and`, `or` | Register-register ALU operations       |
| I-type  | `addi`                 | Register-immediate ALU operation        |
| Load    | `ld`                   | 64-bit load from memory                 |
| Store   | `sd`                   | 64-bit store to memory                  |
| Branch  | `beq`                  | Branch if equal (ALU subtract + zero flag) |

## Architecture

The processor follows the classic textbook single-cycle datapath:

```
                ┌────────────┐
   pc_next ───► │     PC     │──► pc_out ──► Instruction Memory ──► instr
                └────────────┘                                        │
                                                                       ▼
       ┌────────────────────────── Control Unit (opcode) ─────────────┤
       │                                                               │
       ▼                                                               ▼
  Register File ◄── write-back data              Immediate Generator (sign-extend)
       │                                                               │
       ├── read_data1 ────────────────┐                                │
       └── read_data2 ──┬─────────────┼── ALUSrc mux ──────────────────┘
                         │             ▼
                         │            ALU ──► alu_result, zero_flag
                         │             │
                         ▼             ▼
                    Data Memory ◄── address / write_data
                         │
                         ▼
              MemtoReg mux ──► write_back_data ──► Register File
```

- `PC + 4` and the branch target (`PC + immediate`) are muxed based on `Branch & zero_flag` to compute the next PC.
- The **ALU Control** unit decodes `ALUOp` together with `funct3`/`funct7` to select the exact ALU operation.
- The **ALU** itself is built structurally from a chain of gate-level full adders (with an invert + carry-in trick for subtraction), rather than using Verilog's `+`/`-` operators.

## Project Structure

```
.
├── processor.v            # Top-level module wiring all datapath components together
├── pc.v                    # 64-bit program counter
├── instruction_mem.v       # 4096-byte instruction memory (loaded from instructions.txt)
├── control_unit.v          # Opcode decode -> control signals
├── register_file.v         # 32 x 64-bit general-purpose registers
├── immediate_gen.v         # Sign-extended immediate generation (I/S/B formats)
├── alu_control.v           # ALUOp + funct3/funct7 -> ALU control signal
├── alu.v                   # Gate-level ALU (add/sub/and/or), full adder building block
├── data_memory.v           # 1024-byte byte-addressable data memory
├── instructions.txt        # Sample RV64I program (hex bytes) loaded into instruction memory
├── seq_tb.v                 # Top-level testbench: runs the processor to completion
├── pc_tb.v                  # Unit testbench for pc.v
├── instruction_mem_tb.v     # Unit testbench for instruction_mem.v
├── control_unit_tb.v        # Unit testbench for control_unit.v
├── register_file_tb.v       # Unit testbench for register_file.v
├── immediate_gen_tb.v       # Unit testbench for immediate_gen.v
├── alu_control_tb.v         # Unit testbench for alu_control.v
├── alu_tb.v                 # Unit testbench for alu.v
├── data_memory_tb.v         # Unit testbench for data_memory.v
├── team_info.txt            # Team roster
└── IPA_PROJECT_report.pdf   # Full project report
```

## Getting Started

### Requirements

- [Icarus Verilog](https://steveicarus.github.io/iverilog/) (`iverilog` / `vvp`), or any equivalent Verilog simulator (ModelSim, Vivado, Verilator, etc.)

### Running the full processor

The top-level testbench (`seq_tb.v`) instantiates the processor, applies a reset, loads `instructions.txt` into instruction memory, and runs until the program halts (PC stops advancing, a zero instruction is fetched, or a 100,000-cycle safety limit is hit).

```bash
iverilog -o seq_sim seq_tb.v
vvp seq_sim
```

This produces `register_file.txt`, containing the final value of all 32 registers followed by the total cycle count — useful for verifying program output against expected results.

### Running an individual module testbench

Each datapath component has its own standalone testbench. For example, to test just the ALU:

```bash
iverilog -o alu_sim alu.v alu_tb.v
vvp alu_sim
```

Repeat the same pattern (`iverilog -o <name>_sim <module>.v <module>_tb.v && vvp <name>_sim`) for `pc`, `instruction_mem`, `control_unit`, `register_file`, `immediate_gen`, `alu_control`, and `data_memory`.

### Writing your own program

Edit `instructions.txt` with the hex bytes of your RV64I machine code (one byte per line, big-endian instruction order), then re-run the top-level testbench. Only `add`, `sub`, `and`, `or`, `addi`, `ld`, `sd`, and `beq` are currently decoded by the control unit.

## Design Notes

- **Single-cycle limitation**: because every instruction must complete in one clock period, the cycle time is bounded by the slowest instruction (`ld`, which touches the full fetch → decode → execute → memory → write-back chain). This is the main motivation for moving to a pipelined design in a later phase.
- **Big-endian memory layout**: both instruction and data memory are modeled as byte arrays and assembled/disassembled in big-endian order.
- **Structural ALU**: addition and subtraction are built from 64 chained `full_adder` primitives rather than behavioral `+`/`-`, closely mirroring real hardware carry propagation.

## Team

- Chandini Gayathri (2024102020)
- Veekshita Sai (2024102026)
- Naga Rama Hari Kumar (2024102045)

IIIT Hyderabad — Introduction to Processor Architecture, Spring 2026

See `IPA_PROJECT_report.pdf` for the full design report, including block diagrams and per-module verification details.
