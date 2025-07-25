OSMX64 is a 64-bit multi-core SoC architecture designed to be fast, efficient, and low-cost.

## Registers

Registers store data which can be accessed at any time. These include special registers for

| Name       | Size     | Purpose                             |
| ---------- | -------- | ----------------------------------- |
| `x0`       | 64 bits  | Read-only, always `0`               |
| `x1`-`x15` | 64 bits  | General purpose, read-write         |
| `f0`-`f7`  | 32 bits  | IEEE 754 single-precision number    |
| `d0`-`d7`  | 64 bits  | IEEE 754 double-precision number    |
| `v0`-`v7`  | 512 bits | SIMD vector data                    |
| `pc`       | 64 bits  | Program counter, `0` on startup     |
| `sp`       | 64 bits  | Stack pointer                       |
| `fp`       | 64 bits  | Frame pointer                       |
| `ptp`      | 64 bits  | Page tree pointer                   |

## Link Registers

The link registers are for storing specific return addresses from a subroutine

| Name       | Size     | Purpose                                           |
| ---------- | -------- | ---------------------------
| `blr`      | 64 bits  | Branch Link Register      |
| `ilr`      | 64 bits  | Interrupt Link Register   |

When calling a subroutine, it will need to know where to return to
when the execution finishes. To achieve this, we use the branch link
register (BLR). In situations where an interrupt has occurred on the
system, the CPU would need to know where to return to from the interrupt
context, in cases such as this, we'd use the Interrupt Link Register (ILR).

### Internal Registers

Only accessed by the CPU for certain instructions. Cannot be directly read/written.

| Name  | Size    | Purpose                     |
| ----- | ------- | --------------------------- |
| `rp`  | 64 bits | Return pointer for branches |
| `ins` | 64 bits | Current instruction at `pc` |

## Special Registers

OSMX64 provides "special" control registers that may vary depending on the chip
revision. These registers typically include chip-specific mode/control bits to
configure the processor for certain modes of operations, etc. These registers
cannot be accessed via the "mov" instruction and instead rely on independent wrs/rrs
(i.e., special register write / special register read) instructions.

### Common Special Registers

| Name      | Size    | Purpose                     |
| --------- | ------- | --------------------------- |
| `SR_STATE`| 64 bits | Processor state bits        |


### SR\_STATE

| Bit  | Meaning               | Purpose
|------|---------------------- | ---------------------- |
| 0    | Reserved              | Must be zero           |
| 1    | Supervisor            | Supervisor mode enable |
| 2    | Carry                 | Arithmetic carry flag  |
| 3-31 | Reserved (must be 0)  | Must be zero           |


#### Supervisor bit
The supervisor bit may only be writable in a privileged mode of execution
and is therefore readonly in user contexts (writes ignored). It may be unset
by system software to transition the processor into a user context. This bit
may only transition from a 0 to a 1 during an interrupt/trap, therefore manually
writing a 1 to this bit is ignored by the processor.

#### Carry bit
Indicates whether an arithmetic operation has resulted in a
carry/borrow.

## Instructions


### Special Register Access

Special registers are identified with a 16-bit Special Register ID (SRID)
and can be accessed through two special instructions.

| Mnemonic                         | Effect                  |
| -------------------------------- | ----------------------- |
| `wrs` srid: r/imm, val: r        | `SREGS[srid]` = `val`   |
| `rrs` dest: r, srid: r/imm       | `dest` = `SREGS[srid]`  |


### Bitwise Instructions

Bitwise logic operations can write to all registers except `v0`-`v7`, `x0` and `pc`. Can read from all registers except `v0`-`v7`.

| Mnemonic                         | Effect                          |
| -------------------------------- | ------------------------------- |
| `and` dst: r, reg: r, val: r/imm | `dst` = `dst` & `val`           |
| `or` dst: r, reg: r, val: r/imm  | `dst` = `dst` \| `val`          |
| `xor` dst: r, reg: r, val: r/imm | `dst` = `dst` ^ `val`           |
| `mrob` dst: r: mask [bit]: imm   | Fill byte of 'dst' with 'mask'  |
| `mrow` dst: r: mask [bit]: imm   | Fill word of 'dst' with 'mask'  |
| `mrod` dst: r: mask [bit]: imm   | Fill dword of 'dst' with 'mask' |
| `mroq` dst: r: mask [bit]: imm   | Fill qword of 'dst' with 'mask' |

#### Example
```
/* Set x1 to 5 */
mov x1, #5
/* Ands x1 (which equals 5) with 1 */
and x1, x1, #1
/* x1 now equals 1 */
```

### Data Movement Instructions

| Mnemonic                              | Effect
| ------------------------------------- | ---------------------------------- |
| `mov` dst: r, src: r/imm              | `dst` = `src`                      |
| `stb` src: r, [dst: r/m, offset: r/imm] | 1-byte store to memory at `dst`  |
| `stw` src: r, [dst: r/m, offset: r/imm] | 2-byte store to memory at `dst`  |
| `std` src: r, [dst: r/m, offset: r/imm] | 4-byte store to memory at `dst`  |
| `stq` src: r, [dst: r/m, offset: r/imm] | 8-byte store to memory at `dst`  |
| `ldb` dst: r, [src: r/m, offset: r/imm] | 1-byte load from memory at `src` |
| `ldw` dst: r, [src: r/m, offset: r/imm] | 2-byte load from memory at `src` |
| `ldd` dst: r, [src: r/m, offset: r/imm] | 4-byte load from memory at `src` |
| `ldq` dst: r, [src: r/m, offset: r/imm] | 8-byte load from memory at `src` |

#### Example
```
/* Set x1 to 7 */
mov x1, #7

/* Write to memory pointed by sp */
stq x1, [sp]

/* Write to memory pointed by sp + 8 */
stq x1, [sp, #8]

/* Write to memory pointed by sp + x2 */
mov x2, #16
stq x1, [sp, x2]
```

### Arithmetic Instructions

Arithmetic operations can write to all registers except `v0`-`v7`, `x0` and `pc`. Can read from all registers except `v0`-`v7`.

| Mnemonic                             | Effect                |
| ------------------------------------ | --------------------- |
| `add` dst: r, val: r/imm             | `dst` = `dst` + `val` |
| `sub` dst: r, val: r/imm             | `dst` = `dst` - `val` |
| `mul` dst: r, val: r/imm             | `dst` = `dst` * `val` |
| `div` dst: r, val: r/imm             | `dst` = `dst` / `val` |
| `inc` dst: r                         | `dst` = `dst` + `1`   |
| `dec` dst: r                         | `dst` = `dst` - `1`   |

#### Example
```
/* Set x1 to 5 */
mov x1, #5
/* Increment x1 */
inc x1
/* x1 now equals 6 */
```

### Control Flow Instructions

Control flow instructions are used to control which instructions the CPU executes, allowing for the concept of procedures and conditional execution.

| Mnemonic                               | Effect                                          |
| -------------------------------------- | ----------------------------------------------- |
| `hlt`                                  | Execution halted                                |
| `br` dst: r                            | `pc` = `dst`                                    |
| `brl` dst: r                           | `rp` = `pc` + size of opcode, then `pc` = `dst` |
| `bret`                                 | `pc` = `rp`                                     |
| `beq` a: r/m, b: r/m/imm, dst: r/m/imm | Iff `a` = `b`, `pc` = `dst`                     |
| `bne` a: r/m, b: r/m/imm, dst: r/m/imm | Iff `a` != `b`, `pc` = `dst`                    |
| `blt` a: r/m, b: r/m/imm, dst: r/m/imm | Iff `a` < `b`, `pc` = `dst`                     |
| `bgt` a: r/m, b: r/m/imm, dst: r/m/imm | Iff `a` > `b`, `pc` = `dst`                     |
| `ble` a: r/m, b: r/m/imm, dst: r/m/imm | Iff `a` <= `b`, `pc` = `dst`                    |
| `bge` a: r/m, b: r/m/imm, dst: r/m/imm | Iff `a` >= `b`, `pc` = `dst`                    |

#### Example
```
/* Subtract x2 from x1 */
sub x1, x1, x2
beq x1, #0, zero
mov x1, #2
zero:
mov x1, #3
/* Iff x1 - x2 = 0, x1 equals 3 */
/* Iff x1 - x1 != 0, x1 equals 2 */
```

### Opcode list

- `NOP`: `0x00`
- `ADD`: `0x01`
- `SUB`: `0x02`
- `MUL`: `0x03`
- `DIV`: `0x04`
- `INC`: `0x05`
- `DEC`: `0x06`
- `OR`:  `0x07`
- `XOR`: `0x08`
- `AND`: `0x09`
- `NOT`: `0x0A`
- `SLL`: `0x0B`
- `SRL`: `0x0C`
- `MOV_IMM`: `0x0D`
- `HLT`: `0x0E`
- `BR`: `0x0F`
- `MROB`: `0x10`
- `MROW`: `0x11`
- `MROD`: `0x12`
- `MROQ`: `0x13`

Copyright (c) 2024 Quinn Stephens and Ian Marco Moffett.
All rights reserved.
