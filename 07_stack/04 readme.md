# 04.asm – Stack Explanation (32-bit)

## Program

```nasm
BITS 32
section .text
    global _start

_start:
    call work

    mov eax, 1
    xor ebx, ebx
    int 0x80

work:
    push ebp
    mov ebp, esp
    sub esp, 8

    mov dword [ebp - 4], 100
    mov dword [ebp - 8], 200

    mov eax, [ebp - 4]
    mov ebx, [ebp - 8]
    nop

    mov esp, ebp
    pop ebp
    ret
```

## What is different about this program

This is a **32-bit** program. It uses the 32-bit registers `ESP` (stack pointer) and `EBP` (base pointer) instead of `RSP` and `RBP`. Because registers are 32 bits (4 bytes) wide:

- each `push` or `pop` moves ESP by **4 bytes** (not 8),
- `call` pushes a **4-byte** return address,
- each local variable takes **4 bytes** (`dword`).

It exits using `int 0x80` with `eax = 1`, which is the 32-bit way of calling the exit system call.

## Quick idea of how the stack works here

- The stack grows **downward**. `push` makes ESP **smaller** and `pop` makes ESP **bigger**, by 4 bytes each time.
- **ESP** always points to the **top** of the stack.
- **EBP** is the **base pointer**. It stays fixed during the function, so local variables can always be found at `[ebp - 4]`, `[ebp - 8]`, and so on, even while ESP moves.
- `call` automatically **pushes the return address** onto the stack, and `ret` automatically **pops it** and jumps back.
- The program builds a **stack frame** for the function `work`.

In this README, `ESP (start)` means the value ESP has when the program begins.

## What happens from start to end

### 1. Program starts
Execution begins at `_start`. ESP is at `ESP (start)`. Nothing from our program is on the stack yet.

### 2. `call work`
`call` does two things:
1. It pushes the **return address** (the address of `mov eax, 1`, the next instruction) onto the stack. ESP goes down by 4 (`ESP (start) - 4`).
2. It jumps to `work`.

```
Top  ->  return address
```

### 3. `push ebp`
This saves the caller's EBP so it can be restored later. ESP goes down by 4 (`ESP (start) - 8`).

```
Top  ->  saved EBP
         return address
```

### 4. `mov ebp, esp`
EBP now copies ESP, so EBP points at the saved EBP. This is the **base of the stack frame**. ESP does not change (`ESP (start) - 8`).

### 5. `sub esp, 8`
This **reserves 8 bytes** for local variables (two 4-byte values) by moving ESP down by 8 (`ESP (start) - 16`). Nothing is stored yet; the space is only made available.

```
         saved EBP         <- EBP points here
         return address
Top  ->  [8 bytes reserved for local variables]
```

### 6. `mov dword [ebp - 4], 100` and `mov dword [ebp - 8], 200`
These store values in the reserved space, using EBP as the reference.
- `100` goes at `[ebp - 4]`, the first local variable.
- `200` goes at `[ebp - 8]`, the second local variable.

These are normal writes to memory, **not** `push` instructions, so ESP does not change (it stays at `ESP (start) - 16`).

```
Higher addresses
   return address
   saved EBP         <- EBP
   100               <- [ebp - 4]
   200               <- [ebp - 8]    <- ESP (top)
Lower addresses
```

### 7. `mov eax, [ebp - 4]` and `mov ebx, [ebp - 8]`
The values are **read** from the stack into registers: `eax = 100` and `ebx = 200`. Reading does not remove them and ESP does not change.

### 8. `nop`
Does nothing. It marks the point where the frame is fully set up: `eax = 100`, `ebx = 200`, ESP at `ESP (start) - 16`.

### 9. `mov esp, ebp`
This **releases the local variable space** by moving ESP back up to where EBP points. ESP is now `ESP (start) - 8`. The two local values are no longer part of the stack, but are still in memory.

### 10. `pop ebp`
This takes the saved EBP off the top of the stack and puts it back in EBP, **restoring the caller's EBP**. ESP goes up by 4 (`ESP (start) - 4`).

```
Top  ->  return address
```

### 11. `ret`
`ret` pops the return address off the top of the stack and jumps to it. ESP goes up by 4 and is back to `ESP (start)`. Execution continues at `mov eax, 1` in `_start`.

### 12. Exit
`mov eax, 1`, `xor ebx, ebx` and `int 0x80` call the exit system call with code 0. The program ends.

## How ESP changed

| Instruction | ESP | What changed on the stack |
|---|---|---|
| Start | ESP (start) | Empty |
| `call work` | ESP (start) − 4 | Return address pushed |
| `push ebp` | ESP (start) − 8 | Old EBP saved |
| `mov ebp, esp` | ESP (start) − 8 | No change to ESP. EBP now marks the frame |
| `sub esp, 8` | ESP (start) − 16 | 8 bytes reserved for locals |
| `mov [ebp-4], 100` | ESP (start) − 16 | 100 stored. ESP unchanged |
| `mov [ebp-8], 200` | ESP (start) − 16 | 200 stored. ESP unchanged |
| `mov eax/ebx, [...]` | ESP (start) − 16 | Values read. ESP unchanged |
| `mov esp, ebp` | ESP (start) − 8 | Locals released |
| `pop ebp` | ESP (start) − 4 | Old EBP restored |
| `ret` | ESP (start) | Return address popped. Stack back to start |

## Observations from running the program

I assembled it with `nasm -f elf32`, linked it with `ld -m elf_i386`, ran it, and stepped through it in GDB using `stepi`, `info registers esp ebp` and `x/4wx $esp`.

- The program prints nothing and exits normally with code `0`.
- ESP changed in steps of **4 bytes** (not 8), because this is a 32-bit program.
- `call` lowered ESP by 4 with no `push` instruction, because it pushes the return address itself. `ret` raised ESP by 4 for the same reason.
- `sub esp, 8` reserved room for two locals in one step without storing anything.
- The stores to `[ebp - 4]` and `[ebp - 8]` did **not** change ESP. Writing into reserved stack space is different from pushing.
- Examining memory in GDB after the stores showed, from the top of the stack upward: `200` (at ESP), `100`, the saved EBP, and the return address.
- After the two `mov` loads, `eax` was `100` and `ebx` was `200`.
- `mov esp, ebp` followed by `pop ebp` undid the frame setup in reverse order.
- ESP finished at the same value it started with, so the stack was balanced and `ret` returned to the right place.

## Conclusion

This program builds a stack frame for the function `work` in 32-bit mode. `call` saves the return address, `push ebp` and `mov ebp, esp` set up the frame, and `sub esp, 8` reserves space for two 4-byte locals holding `100` and `200`. At the end, `mov esp, ebp` and `pop ebp` tear the frame down and `ret` returns using the saved address. Because this is 32-bit, ESP moves in units of 4 bytes, and it returns to its starting value before the program exits.