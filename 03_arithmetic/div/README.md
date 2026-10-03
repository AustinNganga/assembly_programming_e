# Division (div)

This folder contains two programs that use the unsigned `div` instruction.

**Important find:** after `div`, the arithmetic flags CF, OF, SF, ZF, AF and PF are **undefined** according to the Intel manual. Unlike `add` or `sub`, `div` does not report its result through the flags. The result is in the registers (quotient and remainder), and the flag values seen in GDB are just whatever the CPU happened to leave behind. They carry no meaning and should not be used by a program.

A division that cannot be done (divisor = 0, or a quotient too big for the destination register) does not set a flag. It raises a divide error exception (#DE), which on Linux kills the program with SIGFPE.

---

## div1.asm (8-bit division, 100 / 7)

### Code

```nasm
; al = quotient, ah = remainder

section .data
    dividend dw 100   ; ax = 100
    divisor  db 7     ; bl = 7

section .text
    global _start

_start:
    mov ax, [dividend]  ; ax = 100
    mov bl, [divisor]   ; bl = 7
    div bl              ; al = 14, ah = 2

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Operation

For `div r/m8`, the CPU divides `AX` by the 8-bit operand. The quotient goes in `AL` and the remainder in `AH`.

```
AX = 100 (0x0064),  BL = 7
100 / 7 = 14 remainder 2     (14 * 7 = 98, 100 - 98 = 2)
AL = 14 (0x0E)  quotient
AH = 2  (0x02)  remainder
AX = 0x020E after the division
```

The quotient (14) fits in 8 bits, so there is no divide error.

### Flags (from GDB at `n_break`)

```
eflags  [ AF IF ]
```

- **Set:** AF, IF
- **Cleared:** CF, PF, ZF, SF, OF

### Explanation

**CF, OF, SF, ZF, PF = 0 (undefined after `div`)**
Intel documents these flags as undefined for `div`. They do not describe the quotient or remainder. For example, ZF is 0 even though the remainder is not related to it, and SF is 0 even though no sign test was done. They happen to read 0 here, but another CPU model could show different values.

**AF (Auxiliary Carry Flag) = 1 (set, undefined)**
AF is also undefined after `div`. The division does not involve a carry from bit 3 to bit 4, so AF = 1 is not a result of the operation. It is just the value this CPU left behind, and it should not be interpreted as meaningful.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not modified by `div`. The operating system sets it for normal user programs, so it appears in GDB.

**What to check instead:** the result of `div` is read from the registers, not the flags: `AL = 14` (quotient) and `AH = 2` (remainder).

---

## div2.asm (16-bit division, 50000 / 300)

### Code

```nasm
; ax = quotient, dx = remainder

section .data
    dividend dw 50000   ; Low word
    highpart dw 0       ; High word (DX=0)
    divisor  dw 300

section .text
    global _start

_start:
    mov ax, [dividend]  ; AX = 50000
    mov dx, [highpart]  ; DX = 0
    mov bx, [divisor]   ; BX = 300
    div bx              ; AX = 166, DX = 200

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Operation

For `div r/m16`, the CPU divides the 32-bit value `DX:AX` by the 16-bit operand. The quotient goes in `AX` and the remainder in `DX`.

```
DX:AX = 0x0000C350 = 50000,  BX = 300
50000 / 300 = 166 remainder 200     (166 * 300 = 49800, 50000 - 49800 = 200)
AX = 166 (0x00A6)  quotient
DX = 200 (0x00C8)  remainder
```

`DX` was set to 0 first because `div` always uses `DX:AX` as the dividend. A leftover value in `DX` would change the dividend and could cause a divide error. The quotient (166) fits in 16 bits, so there is no divide error.

### Flags (from GDB at `n_break`)

```
eflags  [ AF IF ]
```

- **Set:** AF, IF
- **Cleared:** CF, PF, ZF, SF, OF

### Explanation

**CF, OF, SF, ZF, PF = 0 (undefined after `div`)**
These flags are undefined for `div`, as in `div1.asm`. They do not describe the quotient (166) or the remainder (200). Their 0 values should not be read as "no carry", "not zero", and so on.

**AF (Auxiliary Carry Flag) = 1 (set, undefined)**
AF is undefined after `div`. It is not produced by a nibble carry in the division, so it has no meaning. It is the same value seen in `div1.asm`, which suggests this CPU consistently leaves AF set after `div`.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not modified by `div`. It is set by the operating system for user programs.

**Note:** Both programs give the same flags `[AF IF]` even though the divisions are different (8-bit vs 16-bit, different values). This shows the flags are not reporting anything about the division. The quotient and remainder are in the registers: `AX = 166`, `DX = 200`.