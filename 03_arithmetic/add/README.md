# Addition (add)

This folder contains two programs that show how the `add` instruction updates the EFLAGS register.

---

## add1.asm (120 + 10)

### Code

```nasm
section .data
    num1 db 120   ; 01111000b
    num2 db 10    ; 00001010b
    result db 0

section .text
    global _start

_start:
    mov al, [num1]      ; al = 120 (0x78)
    add al, [num2]      ; al = 120 + 10 = 130 (0x82)
    mov [result], al

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

```
    01111000   (120)
  + 00001010   (10)
  ----------
    10000010   (0x82 = 130 unsigned, -126 signed)
```

### Flags (from GDB at `n_break`)

```
eflags  [ PF AF SF IF OF ]
```

- **Set:** PF, AF, SF, OF, IF
- **Cleared:** CF, ZF

### Explanation

**CF (Carry Flag) = 0 (cleared)**
CF is set when an *unsigned* addition produces a carry out of the most significant bit (bit 7 for an 8-bit operation), meaning the result does not fit in 8 bits. The unsigned range of 8 bits is 0 to 255. Since 120 + 10 = 130 is within that range, no carry leaves bit 7, so CF stays 0. From an unsigned point of view, the addition is correct.

**PF (Parity Flag) = 1 (set)**
PF looks only at the **least significant byte** of the result and is set when the number of 1-bits in it is **even**. The result `10000010` contains two 1-bits (bit 7 and bit 1). Two is even, so PF = 1. PF has nothing to do with whether the number itself is odd or even, only with the count of set bits.

**AF (Auxiliary Carry Flag) = 1 (set)**
AF indicates a carry from bit 3 into bit 4 (the boundary between the low and high nibble). It is used for BCD arithmetic. Adding the low nibbles gives `1000 + 1010 = 8 + 10 = 18`, which is greater than 15 (the largest value a nibble can hold, `1111`). The excess carries into bit 4, so AF = 1.

**ZF (Zero Flag) = 0 (cleared)**
ZF is set only when the result of the operation is exactly zero. The result is `0x82` (130), which is not zero, so ZF = 0.

**SF (Sign Flag) = 1 (set)**
SF is a copy of the most significant bit (bit 7) of the result. The result `10000010` has bit 7 = 1, so SF = 1. This means that if the result is interpreted as a signed (two's complement) number, it is negative (`0x82 = -126`).

**OF (Overflow Flag) = 1 (set)**
OF indicates **signed overflow**: the true mathematical result does not fit in the signed 8-bit range (-128 to +127). It is set when two operands of the same sign give a result of the opposite sign. Here:
- `num1 = +120` (positive, bit 7 = 0)
- `num2 = +10` (positive, bit 7 = 0)
- The result has bit 7 = 1, so it reads as negative (-126)

The correct answer, +130, exceeds +127, so the signed result is wrong and OF = 1. Internally, OF is the XOR of the carry *into* bit 7 and the carry *out of* bit 7. Here there is a carry into bit 7 but no carry out of bit 7, so `1 XOR 0 = 1`.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not affected by arithmetic instructions. It is set by the operating system for normal user-mode programs, which is why it appears in GDB regardless of what `add` does.

**Note:** The same bits `10000010` mean 130 unsigned or -126 signed. The CPU sets both CF (unsigned view) and OF (signed view) independently. Here the unsigned result is valid (CF = 0) while the signed result is not (OF = 1).

---

## add2.asm (16-bit addition, 32000 + 500)

### Code

```nasm
section .data
    num1 dw 32000
    num2 dw 500
    result dw 0

section .text
    global _start

_start:
    mov ax, [num1]
    add ax, [num2]       ; AX = num1 + num2
    mov [result], ax

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

```
    0111 1101 0000 0000   (32000 = 0x7D00)
  + 0000 0001 1111 0100   (500   = 0x01F4)
  ---------------------
    0111 1110 1111 0100   (32500 = 0x7EF4)
```

### Flags (from GDB at `n_break`, right after the `add`)

```
eflags  [ IF ]
```

- **Set:** IF
- **Cleared:** CF, PF, AF, ZF, SF, OF

The flags were read at `n_break`, before the final `xor ebx, ebx`. That `xor` would overwrite the flags (it sets ZF and PF because its result is 0), so reading after the whole program runs would show the flags of the `xor`, not of the `add`.

### Explanation

**CF (Carry Flag) = 0 (cleared)**
For a 16-bit operation, CF is set when the unsigned result exceeds 65535 (a carry out of bit 15). Since 32000 + 500 = 32500 fits in 16 bits, there is no carry out of bit 15, so CF = 0.

**PF (Parity Flag) = 0 (cleared)**
PF only looks at the lowest 8 bits of the result, even in a 16-bit operation. The low byte is `0xF4` = `11110100`, which has five 1-bits. Five is odd, so PF = 0.

**AF (Auxiliary Carry Flag) = 0 (cleared)**
AF tracks a carry from bit 3 into bit 4. The low nibbles of the operands are `0x0` (from 0x7D00) and `0x4` (from 0x01F4). 0 + 4 = 4, which is not greater than 15, so no carry occurs and AF = 0.

**ZF (Zero Flag) = 0 (cleared)**
The result `0x7EF4` (32500) is not zero, so ZF = 0.

**SF (Sign Flag) = 0 (cleared)**
SF copies bit 15, the most significant bit of the 16-bit result. `0x7EF4` = `0111 1110 1111 0100` has bit 15 = 0, so SF = 0 and the result is positive when read as signed.

**OF (Overflow Flag) = 0 (cleared)**
The signed 16-bit range is -32768 to +32767. Both operands are positive and the result, +32500, fits in that range and is still positive. Internally there is no carry into bit 15 and no carry out of bit 15, so `0 XOR 0 = 0` and OF = 0.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not changed by `add`. It is set by the operating system for user-mode programs, which is why it shows in GDB.

**Note:** Every arithmetic flag is cleared because 32500 fits in both the unsigned (0 to 65535) and signed (-32768 to 32767) 16-bit ranges. This is the opposite of `add1.asm`, where the result overflowed the signed range.