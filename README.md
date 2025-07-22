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

| Mnemonic                         | Effect                 |
| -------------------------------- | ---------------------- |
| `and` dst: r, reg: r, val: r/imm | `dst` = `dst` & `val`  |
| `or` dst: r, reg: r, val: r/imm  | `dst` = `dst` \| `val` |
| `xor` dst: r, reg: r, val: r/imm | `dst` = `dst` ^ `val`  |

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
| `br` dst: r/m/imm                      | `pc` = `dst`                                    |
| `brl` dst: r/m/imm                     | `rp` = `pc` + size of opcode, then `pc` = `dst` |
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

Copyright (c) 2024 Quinn Stephens and Ian Marco Moffett.
All rights reserved.
