# 06.asm – Stack Explanation (32-bit)

## Program

```nasm
BITS 32
section .text
    global _start

_start:
    mov eax, 0x11223344
    mov ebx, 0x1234

    push eax
    push ebx
    nop

    pop ebx
    pop eax

    mov eax, 1
    xor ebx, ebx
    int 0x80
```

## What is different about this program

This is a **32-bit** program. It uses the 32-bit registers (`EAX`, `EBX`) and the 32-bit stack pointer `ESP`. Because registers are 32 bits (4 bytes) wide:

- each `push` or `pop` moves ESP by **4 bytes** (not 8),
- each value on the stack takes **4 bytes**.

It exits using `int 0x80` with `eax = 1`, which is the 32-bit way of calling the exit system call.

## Quick idea of how the stack works here

- The stack grows **downward**. `push` makes ESP **smaller** and `pop` makes ESP **bigger**, by 4 bytes each time.
- **ESP** always points to the **top** of the stack.
- The stack is **LIFO** (Last In, First Out): the last value pushed is the first value popped.
- A 32-bit `push` always stores a full **4 bytes**, even if the number is small. So `ebx = 0x1234` is stored as `0x00001234`.
- Here the stack is used as **temporary storage** for two register values. The program saves both, then restores them.

In this README, `ESP (start)` means the value ESP has when the program begins.

## What happens from start to end

### 1. Program starts
Execution begins at `_start`. ESP is at `ESP (start)`. Nothing from our program is on the stack yet.

### 2. `mov eax, 0x11223344` and `mov ebx, 0x1234`
The two values are loaded into registers. The stack is not involved, so ESP does not change.

### 3. `push eax`
ESP goes down by 4 (`ESP (start) - 4`) and the value of `eax`, `0x11223344`, is stored at the new top.

```
Top  ->  0x11223344
```

### 4. `push ebx`
ESP goes down by another 4 (`ESP (start) - 8`) and `ebx` is stored on top. Even though `0x1234` is small, a whole 4 bytes are used: `0x00001234`.

```
Top  ->  0x00001234    (from ebx)
         0x11223344    (from eax)
```

### 5. `nop`
Does nothing. It marks the point where both values are on the stack, with `ebx`'s value on top and `eax`'s value below it. ESP is `ESP (start) - 8`.

### 6. `pop ebx`
The top value, `0x00001234`, is copied into `ebx`. Then ESP goes up by 4 (`ESP (start) - 4`).

```
Top  ->  0x11223344
```

### 7. `pop eax`
The next value, `0x11223344`, is copied into `eax`. ESP goes up by 4 and is back to `ESP (start)`.

```
(stack is empty again)
```

### 8. Exit
`mov eax, 1`, `xor ebx, ebx` and `int 0x80` call the exit system call with code 0. The stack is not used here and the program ends.

## Why the values go back to the right registers

The pushes were done in the order **eax, then ebx**. The pops were done in the opposite order, **ebx, then eax**. Because the stack is LIFO, the last value pushed (`ebx`) is the first value popped, so each value goes back into the register it came from. If the pops had been in the same order as the pushes (`pop eax` first), the values would have been swapped.

## How ESP changed

| Instruction | ESP | Stack contents (top first) |
|---|---|---|
| Start | ESP (start) | empty |
| `mov eax, ...` / `mov ebx, ...` | ESP (start) | empty |
| `push eax` | ESP (start) − 4 | eax value |
| `push ebx` | ESP (start) − 8 | ebx value, eax value |
| `pop ebx` | ESP (start) − 4 | eax value |
| `pop eax` | ESP (start) | empty |

Two pushes lowered ESP by 8 bytes (2 × 4) and two pops raised it by 8 bytes, so ESP ended where it began.

## Observations from running the program

I assembled it with `nasm -f elf32`, linked it with `ld -m elf_i386`, ran it, and stepped through it in GDB using `stepi`, `info registers esp eax ebx` and `x/2wx $esp`.

- The program prints nothing and exits normally with code `0`.
- ESP changed in steps of **4 bytes** (not 8), because this is a 32-bit program.
- After both pushes, `x/2wx $esp` showed `0x00001234` at the top and `0x11223344` just below it. This matches the LIFO order.
- `push ebx` used a **full 4 bytes** even though `0x1234` is small. The upper bytes were stored as zeros.
- Viewing the raw bytes with `x/8xb $esp` showed `0x11223344` stored as `44 33 22 11`. This is because x86 is **little-endian**: the least significant byte is stored at the lowest address.
- `push` only copies the value, so `eax` and `ebx` kept their values the whole time.
- After the pops, `eax` and `ebx` held exactly the values they had before, because the pops were in reverse order of the pushes.
- ESP finished at the same value it started with, so the pushes and pops were balanced.

## Conclusion

The program uses the stack to temporarily save two registers. `eax` is pushed first and `ebx` second, so `ebx` is on top. Because of LIFO, `ebx` is popped first and `eax` second, and each value returns to its own register. Each push or pop moves ESP by 4 bytes, and ESP returns to its starting value before the program exits.