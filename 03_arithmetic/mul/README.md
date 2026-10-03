# Multiplication (mul)

This folder contains two programs that use the unsigned `mul` instruction.

**How `mul` affects the flags:**
- **CF and OF** are the only meaningful flags. They are cleared (0) when the upper half of the result is zero, meaning the whole product fits in the lower half. They are set (1) when the upper half is non-zero, meaning the product needs the full double-width result.
- **SF, ZF, AF and PF** are **undefined** according to the Intel manual. They do not describe the product, so their values in GDB should not be interpreted.

Note that CF and OF are always equal after `mul`, and here they do not mean "carry" or "signed overflow" the way they do for `add`. They only report whether the upper half of the result is in use.

---

## mul1.asm (8-bit multiplication, 25 * 10)

### Code

```nasm
section .data
    num1 db 25
    num2 db 10
    result dw 0         ; needs 16 bits for result

section .text
    global _start

_start:
    mov al, [num1]      ; al = first operand
    mul byte [num2]     ; ax = al * num2 (25 * 10 = 250)
    mov [result], ax    ; store result in memory

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Operation

For `mul r/m8`, the CPU multiplies `AL` by the operand and stores the 16-bit product in `AX`. `AH` is the upper half and `AL` is the lower half.

```
AL = 25 (0x19),  num2 = 10 (0x0A)
25 * 10 = 250 = 0x00FA
AH = 0x00  (upper half)
AL = 0xFA  (lower half)
```

### Flags (from GDB at `n_break`)

```
eflags  [ IF ]
```

- **Set:** IF
- **Cleared:** CF, PF, AF, ZF, SF, OF

### Explanation

**CF (Carry Flag) = 0 (cleared)**
After `mul`, CF is cleared when the upper half of the product is zero. The product is 250 (`0x00FA`), so `AH = 0` and the whole result fits in `AL` (the 8-bit unsigned maximum is 255). CF = 0 tells us no information was lost by looking only at `AL`.

**OF (Overflow Flag) = 0 (cleared)**
For `mul`, OF is set and cleared together with CF. `AH` is zero, so OF = 0.

**SF, ZF, AF, PF = 0 (undefined after `mul`)**
Intel documents these flags as undefined for `mul`. They do not describe the product. For example, the 0 in ZF does not mean "the result was non-zero" and the 0 in SF does not reflect the sign of `AL`. They just happen to read 0 on this CPU and should not be relied on.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not modified by `mul`. The operating system sets it for normal user programs, so it appears in GDB.

**Note:** The product 250 is below 256, so this multiplication does not need the upper half of the result. That is why CF and OF are both 0.

---

## mul2.asm (16-bit multiplication, 3000 * 200)

### Code

```nasm
section .data
    num1 dw 3000   ; 0x0BB8 = 1011 10111000
    num2 dw 200    ; 0x00C8 =      11001000
    result dd 0    ; 32-bit result

section .text
    global _start

_start:
    mov ax, [num1]      ; moving the value from num1 to ax
    mul word [num2]     ; DX:AX = AX * num2
    mov [result], ax    ; lower 16 bits
    mov [result+2], dx  ; upper 16 bits

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Operation

For `mul r/m16`, the CPU multiplies `AX` by the operand and stores the 32-bit product in `DX:AX`. `DX` is the upper half and `AX` is the lower half.

```
AX = 3000 (0x0BB8),  num2 = 200 (0x00C8)
3000 * 200 = 600000 = 0x000927C0

binary: 1001 0010 0111 1100 0000
DX = 0x0009  (upper 16 bits)
AX = 0x27C0  (lower 16 bits)
```

The product (600000) is larger than the 16-bit unsigned maximum (65535), so it cannot fit in `AX` alone.

### Flags (from GDB at `n_break`)

```
eflags  [ CF IF OF ]
```

- **Set:** CF, OF, IF
- **Cleared:** PF, AF, ZF, SF

### Explanation

**CF (Carry Flag) = 1 (set)**
After `mul`, CF is set when the upper half of the product is non-zero. Here `DX = 0x0009`, which is not zero, because 600000 exceeds 65535. CF = 1 tells us the lower half (`AX`) is not enough to hold the product, so the result must be read as the full 32-bit value in `DX:AX`.

**OF (Overflow Flag) = 1 (set)**
For `mul`, OF always matches CF. `DX` is non-zero, so OF = 1 as well. This is the same condition as CF, not a separate signed overflow check.

**SF, ZF, AF, PF = 0 (undefined after `mul`)**
These flags are undefined for `mul`. They do not describe the 32-bit product, so their 0 values here have no meaning and should not be interpreted (for example, ZF = 0 is not a statement about the result).

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not modified by `mul`. It is set by the operating system for user-mode programs.

**Note:** Compare this with `mul1.asm`. There the product fit in the lower half, so CF = OF = 0. Here it spills into `DX`, so CF = OF = 1. These two programs together show that CF and OF for `mul` simply report whether the upper half of the result is in use.