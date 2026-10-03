# Subtraction (sub)

This folder contains two programs that show how the `sub` instruction updates the EFLAGS register.

**How `sub` sets the flags:**
- **CF (borrow):** set when the unsigned first operand is smaller than the second, so the subtraction has to borrow from beyond the top bit.
- **OF:** set when the signed result is out of range. For subtraction this can only happen when the operands have different signs and the result's sign differs from the first operand's.
- **SF:** copy of the top bit of the result.
- **ZF:** set when the result is zero.
- **PF:** set when the low byte of the result has an even number of 1-bits.
- **AF:** set when bit 4 borrows from bit 3 (the low nibble of the first operand is smaller than the low nibble of the second).

---

## sub1.asm (8-bit subtraction, 50 - 80)

### Code

```nasm
section .data
    num1 db 50   ; 00110010
    num2 db 80   ; 01010000
    result db 0

section .text
    global _start

_start:
    mov al, [num1]
    sub al, [num2]       ; al = 50 - 80
    mov [result], al

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Operation

```
    00110010   (50)
  - 01010000   (80)
  ----------
    11100010   (0xE2 = 226 unsigned, -30 signed)
```

The true result is -30. In 8 bits, this wraps around to 256 - 30 = 226 (`0xE2`), which is -30 in two's complement.

### Flags (from GDB at `n_break`)

```
eflags  [ CF PF SF IF ]
```

- **Set:** CF, PF, SF, IF
- **Cleared:** ZF, AF, OF

### Explanation

**CF (Carry/Borrow Flag) = 1 (set)**
For `sub`, CF acts as a borrow flag. It is set when the first operand is smaller than the second as unsigned numbers. Here 50 < 80, so the subtraction must borrow from beyond bit 7. The unsigned answer would be negative, which cannot be stored, so the result wraps around to 226 and CF = 1 signals this.

**PF (Parity Flag) = 1 (set)**
PF looks only at the lowest 8 bits of the result. `11100010` contains four 1-bits (bits 7, 6, 5 and 1). Four is even, so PF = 1.

**AF (Auxiliary Carry Flag) = 0 (cleared)**
AF tracks a borrow between bit 3 and bit 4. The low nibbles are `0010` (2) and `0000` (0). 2 - 0 = 2, so no borrow is needed from bit 4, and AF = 0.

**ZF (Zero Flag) = 0 (cleared)**
The result `0xE2` is not zero, so ZF = 0.

**SF (Sign Flag) = 1 (set)**
SF copies bit 7 of the result. `11100010` has bit 7 = 1, so SF = 1. As a signed number, the result is negative (-30), which is the correct answer to 50 - 80.

**OF (Overflow Flag) = 0 (cleared)**
OF indicates signed overflow, meaning the correct signed result does not fit in -128 to +127. Both operands are positive (+50 and +80), and subtracting two numbers of the same sign can never overflow. The result -30 fits in the signed range, so OF = 0.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not changed by `sub`. The operating system sets it for normal user programs, so it appears in GDB.

**Note:** The unsigned view is wrong (CF = 1, since 50 - 80 cannot be represented as an unsigned number), while the signed view is correct (OF = 0, since -30 is valid). This is the opposite of `add1.asm`, where the unsigned result was valid and the signed result was not.

---

## sub2.asm (16-bit subtraction, 1000 - 2000)

### Code

```nasm
section .data
    num1 dw 1000
    num2 dw 2000
    result dw 0

section .text
    global _start

_start:
    mov ax, [num1]
    sub ax, [num2]       ; AX = 1000 - 2000
    mov [result], ax

n_break:
    mov eax, 1
    xor ebx, ebx
    int 0x80
```

### Operation

```
    0000 0011 1110 1000   (1000 = 0x03E8)
  - 0000 0111 1101 0000   (2000 = 0x07D0)
  ---------------------
    1111 1100 0001 1000   (0xFC18 = 64536 unsigned, -1000 signed)
```

The true result is -1000. In 16 bits, this wraps around to 65536 - 1000 = 64536 (`0xFC18`).

### Flags (from GDB at `n_break`)

```
eflags  [ CF PF SF IF ]
```

- **Set:** CF, PF, SF, IF
- **Cleared:** ZF, AF, OF

### Explanation

**CF (Carry/Borrow Flag) = 1 (set)**
1000 is smaller than 2000 as unsigned numbers, so the subtraction borrows from beyond bit 15. The unsigned result would be negative and cannot be stored, so it wraps around to 64536, and CF = 1.

**PF (Parity Flag) = 1 (set)**
PF only checks the lowest 8 bits of the result, even in a 16-bit operation. The low byte of `0xFC18` is `0x18` = `00011000`, which has two 1-bits. Two is even, so PF = 1.

**AF (Auxiliary Carry Flag) = 0 (cleared)**
AF tracks a borrow between bit 3 and bit 4. The low nibbles are `1000` (8, from 0x03E8) and `0000` (0, from 0x07D0). 8 - 0 = 8, so no borrow is needed from bit 4, and AF = 0.

**ZF (Zero Flag) = 0 (cleared)**
The result `0xFC18` is not zero, so ZF = 0.

**SF (Sign Flag) = 1 (set)**
SF copies bit 15, the most significant bit of the 16-bit result. `0xFC18` = `1111 1100 0001 1000` has bit 15 = 1, so SF = 1. As a signed number, the result is negative (-1000), which is correct.

**OF (Overflow Flag) = 0 (cleared)**
Both operands are positive (+1000 and +2000), so subtraction cannot overflow. The result -1000 fits in the signed 16-bit range (-32768 to +32767), so OF = 0.

**IF (Interrupt Enable Flag) = 1 (set)**
IF is not changed by `sub`. It is set by the operating system for user-mode programs.

**Note:** This has the same flag pattern as `sub1.asm` (CF, PF, SF set; ZF, AF, OF cleared) for the same reasons, at 16-bit width. In both cases the first operand is smaller than the second, so the result is negative, the unsigned view borrows (CF = 1), and the signed view is correct (OF = 0).