# AC-AVX-Adder — 32-bit Vectorial Adder/Subtractor in VHDL

**UFRJ — Escola Politécnica — Computer Architecture**

A 32-bit vectorial adder/subtractor implemented in VHDL using **Carry-Lookahead (CLA)** for low latency. A single circuit operates on 4, 8, 16, or 32-bit integer lanes simultaneously, controlled by a 2-bit `vecSize_i` signal — inspired by AVX-512 vectorial arithmetic instructions such as `vpaddb`, `vpaddw`, `vpaddd`.

## Interface

| Signal | Bits | Direction | Description |
|--------|------|-----------|-------------|
| `A_i` | 32 | input | Operand A |
| `B_i` | 32 | input | Operand B |
| `mode_i` | 1 | input | `0` = addition, `1` = subtraction |
| `vecSize_i` | 2 | input | Lane width selector |
| `S_o` | 32 | output | Result |

### `vecSize_i` encoding

| Value | Mode | Lanes |
|-------|------|-------|
| `"00"` | 4-bit integers | 8 simultaneous additions |
| `"01"` | 8-bit integers | 4 simultaneous additions |
| `"10"` | 16-bit integers | 2 simultaneous additions |
| `"11"` | 32-bit integer | 1 addition |

## Architecture

The circuit instantiates **8 × CLA4 blocks** covering bits `[31:0]`. Carry propagation between adjacent blocks is selectively gated by AND gates controlled by a combinational mask derived from `vecSize_i`:

```
cin(i+1) = cout(i) AND mask(i)
```

| `vecSize_i` | Carry mask (gates 1–7) |
|-------------|----------------------|
| `"00"` | `0000000` — all carries blocked |
| `"01"` | `1010101` — carries blocked at byte boundaries |
| `"10"` | `1110111` — carry blocked at the 16-bit midpoint |
| `"11"` | `1111111` — all carries propagate freely |

The mask equations reduce to just 3 distinct combinational expressions:

```vhdl
mask(0) <= vecsize_i(0) or  vecsize_i(1);  -- gates 0, 2, 4, 6
mask(1) <= vecsize_i(1);                    -- gates 1, 5
mask(3) <= vecsize_i(0) and vecsize_i(1);  -- gate 3 (center boundary)
```

## Carry-Lookahead vs Ripple Carry

| Architecture | Gate count | Logic levels | Latency |
|-------------|-----------|-------------|---------|
| Ripple Carry (4-bit) | ~14 | 2n = 8 | 8 × t_g |
| **CLA4 (this project)** | ~22 | **2** (AND+OR) | **2 × t_g** |

For the full 32-bit adder, RCA latency would grow to 64 × t_g. CLA keeps it constant at 2 × t_g regardless of operand width.

## Subtraction

Subtraction uses two's complement: `A − B = A + NOT(B) + 1`.

- `B` is bit-inverted when `mode_i = '1'`: `b_eff <= b_i xor (sub_i & sub_i & sub_i & sub_i)`
- The `+1` is injected via carry-in: `c(0) <= cin_i or sub_i`

Using `OR` (rather than direct assignment) ensures correct carry propagation in chained blocks.

## Repository Structure

```
AC-AVX-Adder/
├── src/
│   ├── cla4.vhd              # 4-bit CLA adder/subtractor block
│   ├── cla4_tb.vhd           # Testbench for cla4 (5 test cases)
│   ├── vec_adder.vhd         # 32-bit top-level (8 × CLA4 + carry gates)
│   ├── vec_adder_tb.vhd      # Testbench for vec_adder (11 test cases)
│   └── vec_adder_full.vhd    # Merged file for Digital simulator
├── digital/
│   └── vec_adder.dig         # Circuit in Digital simulator
└── README.md
```

## How to Simulate

### Requirements

```bash
sudo apt install ghdl
```

### Run testbenches

```bash
# CLA4 block (5 assertions)
ghdl -a src/cla4.vhd
ghdl -a src/cla4_tb.vhd
ghdl -e cla4_tb
ghdl -r cla4_tb

# Full 32-bit adder (11 assertions)
ghdl -a src/cla4.vhd
ghdl -a src/vec_adder.vhd
ghdl -a src/vec_adder_tb.vhd
ghdl -e vec_adder_tb
ghdl -r vec_adder_tb
```

Expected output:
```
src/vec_adder_tb.vhd:106:5:@110ns:(report note): Todos os testes passaram!
```

## Test Cases

| `vecSize` | mode | A | B | Expected S | Purpose |
|-----------|------|---|---|-----------|---------|
| `"00"` | add | `0x12345678` | `0x11111111` | `0x23456789` | nibble-wise addition |
| `"00"` | add | `0xFFFFFFFF` | `0x11111111` | `0x00000000` | carry isolation (4-bit) |
| `"01"` | add | `0x0A0B0C0D` | `0x01020304` | `0x0B0D0F11` | byte-wise addition |
| `"01"` | add | `0xFF0000FF` | `0x01000001` | `0x00000000` | carry isolation (8-bit) |
| `"10"` | add | `0x00010002` | `0x00030004` | `0x00040006` | halfword addition |
| `"10"` | add | `0xFFFF0000` | `0x00010000` | `0x00000000` | carry isolation (16-bit) |
| `"11"` | add | `0xFFFF0001` | `0x0000FFFF` | `0x00000000` | 32-bit overflow |
| `"11"` | add | `0x00000001` | `0x00000001` | `0x00000002` | simple addition |
| `"11"` | sub | `0x00000005` | `0x00000003` | `0x00000002` | 32-bit subtraction |
| `"11"` | sub | `0x00000010` | `0x00000010` | `0x00000000` | zero result |
| `"01"` | sub | `0xFF200F0A` | `0x01100F05` | `0xFE100005` | byte-wise subtraction |

## Digital Simulator

The circuit was validated in [Digital](https://github.com/hneemann/Digital) using the VHDL external component with GHDL backend. Since Digital does not support multiple VHDL files in a single component, `vec_adder_full.vhd` (a concatenation of `cla4.vhd` and `vec_adder.vhd`) is used as the component source.

Component configuration:
- **Inputs:** `A_i:32, B_i:32, mode_i:1, vecSize_i:2`
- **Output:** `S_o:32`
- **Application:** GHDL

## Authors

Luiz Cavalini, Eduardo Viana, Henrique Kezen, Rafael Maurício

## References

- Patterson, D. A. & Hennessy, J. L. *Computer Organization and Design: RISC-V Edition*, 2nd ed., Elsevier, 2021.
- Intel Corporation. *Intel 64 and IA-32 Architectures Software Developer's Manual*, Vol. 2.
- Neemann, H. *Digital — Logic Designer and Circuit Simulator*. https://github.com/hneemann/Digital
- Texas Instruments. *SN74S181 — 4-Bit ALU*. https://www.ti.com/lit/ds/symlink/sn74s181.pdf
- Texas Instruments. *SN74S182 — Look-Ahead Carry Generator*. https://www.ti.com/lit/ds/symlink/sn74s182.pdf
