# 05.asm – Stack Explanation

## Program

```nasm
section .text
    global _start

_start:
    mov rax, 0x1122334455667788
    mov rbx, 0x1234

    push rax
    push rbx
    nop

    pop rbx
    pop rax

    mov rax, 60
    xor rdi, rdi
    syscall
```

## Quick idea of how the stack works here

- The stack grows **downward**. `push` makes RSP **smaller** and `pop` makes RSP **bigger**, by 8 bytes each time.
- **RSP** always points to the **top** of the stack.
- The stack is **LIFO** (Last In, First Out): the last value pushed is the first value popped.
- A 64-bit `push` always stores a full **8 bytes**, even if the number is small. So `rbx = 0x1234` is stored as `0x0000000000001234`.
- Here the stack is used as **temporary storage** for two register values. The program saves both, then restores them.

In this README, `RSP (start)` means the value RSP has when the program begins.

## What happens from start to end

### 1. Program starts
Execution begins at `_start`. RSP is at `RSP (start)`. Nothing from our program is on the stack yet.

### 2. `mov rax, 0x1122334455667788` and `mov rbx, 0x1234`
The two values are loaded into registers: `rax` holds a large 8-byte value and `rbx` holds a small value. The stack is not involved, so RSP does not change.

### 3. `push rax`
RSP goes down by 8 (`RSP (start) - 8`) and the full 8-byte value of `rax`, `0x1122334455667788`, is stored at the new top.

```
Top  ->  0x1122334455667788
```

### 4. `push rbx`
RSP goes down by another 8 (`RSP (start) - 16`) and `rbx` is stored on top. Even though `0x1234` is small, a whole 8 bytes are used: `0x0000000000001234`.

```
Top  ->  0x0000000000001234    (from rbx)
         0x1122334455667788    (from rax)
```

### 5. `nop`
Does nothing. It marks the point where both values are on the stack, with `rbx`'s value on top and `rax`'s value below it. RSP is `RSP (start) - 16`.

### 6. `pop rbx`
The top value, `0x0000000000001234`, is copied into `rbx`. Then RSP goes up by 8 (`RSP (start) - 8`).

```
Top  ->  0x1122334455667788
```

### 7. `pop rax`
The next value, `0x1122334455667788`, is copied into `rax`. RSP goes up by 8 and is back to `RSP (start)`.

```
(stack is empty again)
```

### 8. Exit
`mov rax, 60`, `xor rdi, rdi` and `syscall` call the exit system call with code 0. The stack is not used here and the program ends.

## Why the values go back to the right registers

The pushes were done in the order **rax, then rbx**. The pops were done in the opposite order, **rbx, then rax**. Because the stack is LIFO, the last value pushed (`rbx`) is the first value popped, so each value goes back into the register it came from. If the pops had been in the same order as the pushes (`pop rax` first), the values would have been swapped.

## How RSP changed

| Instruction | RSP | Stack contents (top first) |
|---|---|---|
| Start | RSP (start) | empty |
| `mov rax, ...` / `mov rbx, ...` | RSP (start) | empty |
| `push rax` | RSP (start) − 8 | rax value |
| `push rbx` | RSP (start) − 16 | rbx value, rax value |
| `pop rbx` | RSP (start) − 8 | rax value |
| `pop rax` | RSP (start) | empty |

Two pushes lowered RSP by 16 bytes (2 × 8) and two pops raised it by 16 bytes, so RSP ended where it began.

## Observations from running the program

I assembled it with `nasm -f elf64`, linked it with `ld`, ran it, and stepped through it in GDB using `stepi`, `info registers rsp rax rbx` and `x/2gx $rsp`.

- The program prints nothing and exits normally with code `0`.
- RSP dropped by 8 after each `push` and rose by 8 after each `pop`, which shows the stack grows downward.
- After both pushes, `x/2gx $rsp` showed `0x1234` at the top and `0x1122334455667788` just below it. This matches the LIFO order.
- `push rbx` used a **full 8 bytes** even though `0x1234` is small. The upper bytes were stored as zeros.
- Viewing the raw bytes with `x/16xb $rsp` showed the number `0x1122334455667788` stored as `88 77 66 55 44 33 22 11`. This is because x86 is **little-endian**: the least significant byte is stored at the lowest address.
- `push` only copies the value, so `rax` and `rbx` kept their values the whole time.
- After the pops, `rax` and `rbx` held exactly the values they had before, because the pops were in reverse order of the pushes.
- RSP finished at the same value it started with, so the pushes and pops were balanced.

## Conclusion

The program uses the stack to temporarily save two registers. `rax` is pushed first and `rbx` second, so `rbx` is on top. Because of LIFO, `rbx` is popped first and `rax` second, and each value returns to its own register. Each push or pop moves RSP by 8 bytes, and RSP returns to its starting value before the program exits.