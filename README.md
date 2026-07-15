## System Architecture
## Instruction Set Architecture (ISA)

### 1. RV32M Extension (Hardware Multiply/Divide)
*   **MUL, MULH, MULHU, MULHSU:** 32x32 multiplication returning 64-bit results (truncated to lower/upper 32 bits depending on instruction).
*   **DIV, DIVU, REM, REMU:** 32-bit hardware division and remainder. Fully compliant with RISC-V edge cases (`INT_MIN / -1` overflow, Divide-by-Zero).

### 2. Custom MAU (Opcode: `0001011`)
Uses the standard RISC-V `custom-0` opcode space mapped via `funct3`.
*   `ABS`: Absolute value (1-cycle combinational)
*   `MAX`: Maximum of two registers (1-cycle combinational)
*   `MIN`: Minimum of two registers (1-cycle combinational)
*   `SQRT`: 32-bit integer square root (16-cycle FSM)
*   `LOG2`: Log base 2 via Count Leading Zeros (1-cycle O(1) tree)

---

## Hardware Modules

| Module | File | Description |
| :--- | :--- | :--- |
| **Multiplier** | `multiplier.v` | 3-cycle pipelined 32x32 multiplier mapped to Xilinx DSP48 slices. |
| **Divider** | `divider.v` | 32-cycle restoring algorithm FSM with pre-check for RISC-V edge cases. |
| **MAU Simple** | `mau_simple.v` | Combinational logic for ABS, MAX, MIN. |
| **SQRT Unit** | `sqrt_unit.v` | 16-cycle non-restoring FSM (processes 2 bits/cycle). |
| **CLZ Unit** | `clz_unit.v` | 5-level hierarchical LUT priority encoder for O(1) LOG2. |
| **Execute Stage**| `execute.v` | Extended datapath with result MUX for ALU/MUL/DIV/MAU. |
| **Control** | `opcode.vh` | Decodes RV32IM `funct7` and Custom-0 opcodes. |
| **Peripherals** | `sev_seg.v` | Drivers for  7-segment displays. |

---

## Use Cases & FPGA Validation
The system was validated on the FPGA using 5+ C programs compiled with `-march=rv32im`:
1. **DSP Filtering:** 16-tap FIR filter using `MUL` and `ABS`.
2. **Euclidean Distance:** Calculates $\sqrt{(x_1-x_2)^2 + (y_1-y_2)^2}$ using `SUB`, `MUL`, `ADD`, `SQRT`.
3. **Data Normalization:** Clamping sensor data using `MAX`/`MIN`.
4. **Logarithmic Scaling:** Decibel conversion using `LOG2`.
5. **GCD Calculation:** Recursive division using `DIV`/`REM`.

---

## Resource Utilization (Synthesis Estimates)
*Target: Xilinx Artix-7 XC7A100T (Nexys A7)*

| Resource | Usage | Utilization |
| :--- | :--- | :--- |
| LUTs | ~4,125 | ~6.5% |
| Flip-Flops | ~2,540 | ~2.0% |
| BRAM (36Kb) | 4 | ~3.0% |
| DSP Slices | 3 | ~1.2% |

---

## Setup & Getting Started

### Prerequisites
*   Xilinx Vivado (2022.1 or later recommended)
*   RISC-V GNU Toolchain (`riscv32-unknown-elf-gcc`)

### Compiling C Programs for the Processor
Use the provided Makefile or run the following command to compile a C program into instruction memory hex files:
```bash
riscv32-unknown-elf-gcc -O1 -march=rv32im -mabi=ilp32 -nostdlib -T linker.ld -o program.elf program.c
riscv32-unknown-elf-objcopy -O verilog program.elf program.hex
```

**OR Another Way to generate Hex Files**

Add everything required to generate mem files to makefile
```bash
make <your_c_prog_name>
```
### Simulation
Module-level testbenches (e.g., `tb_multiplier.v`, `tb_divider.v`) can be run using Vivado XSIM or ModelSim. Golden vectors are generated via the included Python scripts:
```bash
python3 gen_vectors.py > input.txt
```

### FPGA Build
1. Open Vivado and create a project targeting the `xc7a100tcsg324-1` FPGA.
2. Add all `.v` and `.vh` files from the `src/` directory.
3. Set `top_module.v` as the top module.
4. Add the provided `Nexys-A7-100T.xdc` constraints file.
5. Run Synthesis, Implementation, and Generate Bitstream.

---
