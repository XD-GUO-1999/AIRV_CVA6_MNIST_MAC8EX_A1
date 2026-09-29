# AIRV CVA6 MAC8EX A1 --- User and Implementation Guide

This guide documents the cleaned **MAC8EX A1** implementation for the
AIRV/CVA6 MNIST accelerator project.

MAC8EX A1 is an execution-stage implementation: the custom instruction
is decoded as a `MULT` operation and executed inside the existing CVA6
multiplier path. It does **not** use the CV-X-IF coprocessor datapath
used by later accelerator versions.

All line numbers in the implementation section refer directly to the
files delivered in `cva6_mac8ex_a1`.

------------------------------------------------------------------------

## 1. MAC8EX A1 Overview

MAC8EX performs eight packed INT8 multiply-accumulate operations.

Conceptually:

``` text
rd =
    old rd
  + input[0] × weight[0]
  + input[1] × weight[1]
  + input[2] × weight[2]
  + input[3] × weight[3]
  + input[4] × weight[4]
  + input[5] × weight[5]
  + input[6] × weight[6]
  + input[7] × weight[7]
```

The instruction explicitly encodes four source-register indices plus
`rd`.

``` text
bits [31:27] -> W4
bits [26:22] -> W3
bits [21:17] -> W2
bits [16:12] -> W1
bits [11:7]  -> rd
bits [6:0]   -> opcode
```

The assembler syntax is:

``` text
mac8ex d,W1,W2,W3,W4
```

The custom opcode is:

``` text
0001011 = 0x0b
```

The five CPU register reads are used as:

``` text
W1      -> first packed input word
W2      -> first packed weight word
rd      -> accumulator
W3      -> second packed input word
W4      -> second packed weight word
```

The exact software convention must remain consistent with this hardware
mapping.

------------------------------------------------------------------------

## 2. Main Execution Path

``` text
mac8ex d,W1,W2,W3,W4
        ↓
GNU assembler / binutils
        ↓
decoder.sv
        ↓
MAC8EX operation
fu = MULT
        ↓
issue_read_operands.sv
        ↓
five GPR values
        ↓
scoreboard.sv
dependency / forwarding
        ↓
issue_stage.sv
        ↓
mult.sv
        ↓
multiplier.sv
        ↓
8-way INT8 MAC
        ↓
multiplier pipeline register
        ↓
rd writeback
```

The key architectural point is that MAC8EX A1 reuses the existing
integer multiplier execution unit instead of offloading the operation to
CV-X-IF.

------------------------------------------------------------------------

## 3. Project Integration

The files in this package are the source files modified for MAC8EX A1.

They must be placed into the corresponding locations of the complete
AIRV/CVA6 project.

The package contains:

``` text
ariane_pkg.sv
cva6.sv
decoder.sv
issue_read_operands.sv
issue_stage.sv
scoreboard.sv
mult.sv
multiplier.sv
rv_i
riscv-opc.h
riscv-opc.c
riscv.h
tc-riscv.c
```

The uploaded modification set does not contain the complete AIRV
repository or the MNIST application source, so this guide does not
invent software-file locations that are not present in the supplied
files.

------------------------------------------------------------------------

## 4. Environment and Validation Workflow

Use the normal AIRV environment setup for the complete project.

Typical checks are:

``` bash
source /path/to/setup.sh

echo $PROJECTROOT
which riscv-none-elf-gcc
which vsim
```

Compile MNIST using the MAC8EX-capable toolchain:

``` bash
cd $PROJECTROOT/sw/app
make clean
make mnist
```

Run RTL simulation:

``` bash
cd $PROJECTROOT
make sim APP=mnist
```

When validating this cleaned version, compare the MNIST output,
instruction count, cycle count, and multiplier-stage waveform with the
known working MAC8EX A1 implementation.

------------------------------------------------------------------------

# 5. Implementation Guide

## 5.1 `ariane_pkg.sv`

### Lines 75--76 --- five GPR read ports

Sets:

``` systemverilog
NR_RGPR_PORTS = 5
```

MAC8EX needs five integer-register values at issue time: four explicitly
encoded source registers plus the previous `rd` value used as the
accumulator.

### Line 448 --- `MAC8EX` operation

Adds `MAC8EX` to the functional-unit operation enumeration.

This identifier follows the instruction from decode into the multiplier
execution path.

### Lines 579--580 --- extra execution operands

Adds:

``` systemverilog
operand_d
operand_e
```

to `fu_data_t`.

These fields carry the fourth and fifth GPR values through the issue
pipeline to `mult.sv`.

------------------------------------------------------------------------

## 5.2 `cva6.sv`

### Lines 164--165 --- register-file read-port configuration

Sets:

``` systemverilog
NrRgprPorts = 5
```

This instantiates the integer register file with enough read ports for
MAC8EX.

------------------------------------------------------------------------

## 5.3 `decoder.sv`

### Lines 1189--1197 --- MAC8EX custom instruction decode

Adds the custom-0 opcode:

``` text
0001011
```

The instruction is sent to:

``` systemverilog
instruction_o.fu = MULT;
instruction_o.op = MAC8EX;
```

`rs1`, `rs2`, and `rd` are extracted directly.

`rd` is both the architectural destination and the source of the
previous accumulator value.

### Lines 1278--1286 --- W3/W4 register-index transport

MAC8EX reuses the `RS3` immediate-selection path to transport two
additional five-bit register indices.

The instruction fields:

``` text
instruction[26:22]
instruction[31:27]
```

are packed into:

``` text
instruction_o.result[4:0]
instruction_o.result[9:5]
```

These fields are later interpreted as the additional GPR addresses.

------------------------------------------------------------------------

## 5.4 `issue_read_operands.sv`

This is the main five-register operand-routing block.

### Lines 43--48 --- rs4/rs5 interface

Adds two additional scoreboard/read-operands channels:

``` text
rs4
rs5
```

### Lines 96--102 --- extra operand storage

Adds the register-file and pipeline storage for:

``` text
operand_d
operand_e
```

### Line 121 --- forwarding control

Adds:

``` text
forward_rs4
forward_rs5
```

so the extra source registers participate in forwarding.

### Lines 134--135 --- execution-data connection

Connects `operand_d_q` and `operand_e_q` into `fu_data_o`.

### Lines 189--198 --- MAC8EX register-address mapping

For MAC8EX:

``` text
rs3 = rd
rs4 = result[4:0]
rs5 = result[9:5]
```

Therefore the old `rd` value becomes the accumulator, while W3 and W4
provide the two additional source-register addresses.

### Lines 239--246 --- accumulator dependency handling

Checks whether the old `rd` value is waiting for a previous write.

If the correct value can be forwarded, it is used. Otherwise issue
stalls.

### Lines 251--260 --- rs4/rs5 dependency handling

Adds RAW-dependency checks for the two extra MAC8EX sources.

Each source can either be forwarded or cause an issue stall until its
value becomes available.

### Lines 272--282 --- operand selection

Adds `operand_d` and `operand_e` to the normal operand-selection logic
and allows MAC8EX to use the GPR third operand as its accumulator.

### Lines 302--307 --- forwarded extra operands

Selects forwarded rs4/rs5 values when required.

### Lines 487--495 --- five-port GPR address packing

The five register-file addresses are packed in this order:

``` text
{
    W4,
    W3,
    rd,
    W2,
    W1
}
```

The corresponding read data are therefore:

``` text
rdata[0] -> W1
rdata[1] -> W2
rdata[2] -> rd / accumulator
rdata[3] -> W3
rdata[4] -> W4
```

### Lines 608--610 --- map extra read ports

Maps:

``` text
rdata[3] -> operand_d
rdata[4] -> operand_e
```

### Lines 620--621 --- reset extra operand registers

Resets the additional operand pipeline registers.

### Lines 633--634 --- pipeline extra operands

Registers `operand_d_n` and `operand_e_n` together with the normal
issue-stage operands.

------------------------------------------------------------------------

## 5.5 `issue_stage.sv`

### Lines 100--101 --- XLEN-wide accumulator source

The third GPR source uses XLEN width when the five-port configuration is
active.

This is required because MAC8EX uses the old `rd` value as a full
accumulator.

### Lines 117--124 --- additional source channels

Declares the rs4 and rs5 address/data/valid signals.

### Lines 163--170 --- scoreboard wiring

Connects rs4 and rs5 to `scoreboard.sv`.

### Lines 215--221 --- read-operands wiring

Connects the same channels to `issue_read_operands.sv`.

------------------------------------------------------------------------

## 5.6 `scoreboard.sv`

### Lines 44--50 --- rs4/rs5 interface

Adds the two extra source-register interfaces to the scoreboard.

### Lines 319--322 --- forwarding request state

Adds forwarding-request and validity state for rs4 and rs5.

### Lines 335--337 --- writeback forwarding checks

Checks whether current writeback ports contain values required by W3 or
W4.

### Lines 352--354 --- in-flight dependency checks

Checks scoreboard entries that have been issued but have not yet written
their result.

### Lines 371--372 --- extra-source validity

Generates valid signals for rs4 and rs5 while excluding x0 from real
dependencies.

### Lines 432--469 --- forwarding arbiters

Adds the two forwarding arbiters:

``` text
i_sel_rs4
i_sel_rs5
```

They select the newest available forwarded values for the two additional
MAC8EX source registers.

------------------------------------------------------------------------

## 5.7 `mult.sv`

### Lines 29--31 --- route MAC8EX to the multiplier

Adds `MAC8EX` to `mul_valid_op`.

This is the key difference between this A1 implementation and a CV-X-IF
implementation: MAC8EX is handled directly by the normal multiplier
functional unit.

### Lines 53--60 --- multiplier operand connection

The multiplier receives:

``` text
operand_a_i = operand_a
operand_b_i = operand_b
operand_c_i = imm
operand_d_i = operand_d
operand_e_i = operand_e
```

For MAC8EX these correspond to the four packed source words plus the
accumulator.

------------------------------------------------------------------------

## 5.8 `multiplier.sv`

This file contains the actual eight-way MAC datapath.

### Lines 28--33 --- five multiplier operands

The multiplier interface contains:

``` text
operand_a_i
operand_b_i
operand_c_i
operand_d_i
operand_e_i
```

`operand_c_i` is the accumulator.

`operand_d_i` and `operand_e_i` form the second packed input/weight
pair.

### Lines 79--90 --- eight-lane INT8 MAC calculation

Defines:

``` systemverilog
mac8ex_res_d
```

The datapath performs four byte-wise products from
`operand_a_i × operand_b_i` and four more from
`operand_d_i × operand_e_i`.

The eight products are summed with:

``` text
operand_c_i
```

as the previous accumulator.

The input-byte expressions use a leading zero bit before signed
multiplication:

``` systemverilog
$signed({1'b0, input_byte})
```

while weight bytes are interpreted with signed arithmetic.

### Line 101 --- MAC8EX multiplier-valid support

Adds `MAC8EX` to the set of operations accepted by the multiplier
pipeline.

### Lines 133--143 --- result selection

When the registered operation is `MAC8EX`, the multiplier output
selects:

``` systemverilog
mac8ex_res_q
```

### Lines 160--174 --- MAC8EX pipeline register

Resets and registers `mac8ex_res_q` together with the existing
multiplier pipeline state.

Therefore MAC8EX follows the same one-stage registered result structure
as the multiplier unit.

------------------------------------------------------------------------

## 5.9 `rv_i`

### Lines 27--28 --- custom opcode definition

Defines:

``` text
mac8ex  6..2=0x02 1..0=3
```

which produces opcode:

``` text
0x0b
```

The remaining instruction fields are variable and contain `rd` plus
W1--W4.

------------------------------------------------------------------------

## 5.10 `riscv-opc.h`

### Lines 24--26 --- instruction match and mask

Defines:

``` c
MATCH_MAC8EX 0xb
MASK_MAC8EX  0x7f
```

Only the seven opcode bits are fixed because the other fields carry
register indices.

### Line 2789 --- instruction declaration

Registers:

``` c
DECLARE_INSN(mac8ex, MATCH_MAC8EX, MASK_MAC8EX)
```

with the GNU RISC-V opcode infrastructure.

------------------------------------------------------------------------

## 5.11 `riscv-opc.c`

### Lines 322--323 --- assembler mnemonic

Adds:

``` text
mac8ex
```

with operand format:

``` text
d,W1,W2,W3,W4
```

This tells GAS that MAC8EX contains one destination/accumulator register
and four custom register fields.

------------------------------------------------------------------------

## 5.12 `riscv.h`

### Lines 109--128 --- custom register field helpers

Defines extraction and encoding helpers for W1--W4.

The bit positions are:

``` text
W1 -> shift 12
W2 -> shift 17
W3 -> shift 22
W4 -> shift 27
```

Each field is five bits wide and stores a normal GPR index.

These macros must remain consistent with the instruction layout expected
by `decoder.sv`.

------------------------------------------------------------------------

## 5.13 `tc-riscv.c`

### Lines 1392--1404 --- validate W1--W4 fields

Adds the custom GAS operand type:

``` text
W
```

with subfields:

``` text
W1
W2
W3
W4
```

The validation logic marks the corresponding instruction bits as
occupied.

### Lines 3308--3332 --- parse W1--W4 register operands

Parses each W operand as a normal GPR name and inserts its register
number into the custom instruction field using:

``` text
ENCODE_MAC8_RS1
ENCODE_MAC8_RS2
ENCODE_MAC8_RS3
ENCODE_MAC8_RS4
```

This allows assembly such as:

``` text
mac8ex rd, rsA, rsB, rsC, rsD
```

using normal RISC-V register names.

------------------------------------------------------------------------

# 6. Complete Register Mapping

The complete path is:

``` text
Assembly W1
    -> instruction[16:12]
    -> decoder rs1
    -> register-file rdata[0]
    -> operand_a
    -> multiplier operand_a_i

Assembly W2
    -> instruction[21:17]
    -> decoder rs2
    -> register-file rdata[1]
    -> operand_b
    -> multiplier operand_b_i

Assembly rd
    -> instruction[11:7]
    -> register-file rdata[2]
    -> imm / accumulator
    -> multiplier operand_c_i
    -> final destination rd

Assembly W3
    -> instruction[26:22]
    -> decoder result[4:0]
    -> register-file rdata[3]
    -> operand_d
    -> multiplier operand_d_i

Assembly W4
    -> instruction[31:27]
    -> decoder result[9:5]
    -> register-file rdata[4]
    -> operand_e
    -> multiplier operand_e_i
```

This mapping must remain consistent across the toolchain, decoder,
register-file read logic, scoreboard, issue stage, `mult.sv`, and
`multiplier.sv`.

------------------------------------------------------------------------

# 7. Recommended Debugging Order

If MAC8EX produces an incorrect result, debug in this order:

``` text
1. Check the generated disassembly
   -> verify mac8ex and W1–W4

2. Check decoder.sv
   -> opcode, fu=MULT, op=MAC8EX

3. Check instruction_o.result[9:0]
   -> verify W3/W4 indices

4. Check issue_read_operands.sv
   -> verify five GPR addresses

5. Check scoreboard.sv
   -> verify stalls and forwarding

6. Check fu_data operands
   -> operand_a/b/imm/d/e

7. Check mult.sv
   -> MAC8EX enters multiplier path

8. Check multiplier.sv
   -> eight byte products and accumulator

9. Check mac8ex_res_d / mac8ex_res_q

10. Check final rd writeback
```

Useful waveform signals include:

``` text
operation_i
operand_a_i
operand_b_i
operand_c_i
operand_d_i
operand_e_i
mac8ex_res_d
mac8ex_res_q
mult_valid_i
mult_valid_o
result_o
```

------------------------------------------------------------------------

# 8. Validation Notes

The cleanup performed for this package is intentionally conservative.

The functional tokens of all 13 supplied source files were checked
against the uploaded MAC8EX A1 files after removing comments and
whitespace. The cleanup changes comments, formatting, and removes
obsolete commented debug code; it does not intentionally change the
MAC8EX encoding, register mapping, dependency logic, arithmetic, or
multiplier pipeline behavior.

A complete RTL regression was not run from this package because the
uploaded archive contains only the modified source files, not a complete
standalone AIRV/CVA6 build tree.

After integrating the files into the full project, run the normal MNIST
simulation before using the cleaned version as the frozen A1 release.
