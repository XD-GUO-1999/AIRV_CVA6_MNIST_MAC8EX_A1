# AIRV CVA6 MAC8EX A1 Accelerator

Hardware/software co-design for accelerating a quantized MNIST CNN on
the CVA6 RISC-V processor.

This implementation includes:

-   Custom RISC-V instruction: `MAC8EX`
-   8-way INT8 multiply-accumulate
-   Direct integration into the CVA6 `MULT` execution path
-   Five GPR read ports
-   `rd`-based accumulation
-   Four explicitly encoded source registers: `W1`, `W2`, `W3`, `W4`
-   Extended scoreboard dependency checking and forwarding
-   Modified multiplier datapath
-   Modified GNU assembler/toolchain support
-   Questa RTL simulation support

## Documentation

For the MAC8EX A1 architecture, instruction encoding, register mapping,
multiplier integration, toolchain changes, debugging procedure, and
exact source-code line locations:

👉 [User and Implementation Guide](USER_GUIDE_MAC8EX_A1.md)

## Source Code

The cleaned MAC8EX A1 source files are located in:

``` text
cleaned_code/
```

## Accelerator Overview

``` text
mac8ex d,W1,W2,W3,W4
        ↓
GNU assembler
        ↓
CVA6 decoder
        ↓
five-register read path
        ↓
scoreboard / forwarding
        ↓
MULT functional unit
        ↓
8-way INT8 MAC
        ↓
rd writeback
```

Unlike later CV-X-IF accelerator versions, MAC8EX A1 executes directly
inside the CVA6 multiplier path.

## Instruction Encoding

``` text
bits [31:27] -> W4
bits [26:22] -> W3
bits [21:17] -> W2
bits [16:12] -> W1
bits [11:7]  -> rd
bits [6:0]   -> 0x0b
```

For exact line numbers and the purpose of each source modification, see
`USER_GUIDE_MAC8EX_A1.md`.
