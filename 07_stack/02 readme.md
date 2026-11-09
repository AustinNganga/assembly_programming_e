# 02.asm – Stack Explanation

## Program

```nasm
section .text
    global _start

_start:
    call work
    mov rax, 60
    xor rdi, rdi
    syscall

work:
    push rbp
    mov rbp, rsp
    sub rsp, 16

    mov qword [rbp - 8], 100
    mov qword [rbp - 16], 200

    mov rax, [rbp - 8]
    mov rbx, [rbp - 16]
    nop

    mov rsp, rbp
    pop rbp
    ret
```

## Quick idea of how the stack works here

- The stack grows **downward**. `push` makes RSP **smaller** and `pop` makes RSP **bigger**, by 8 bytes each time.
- **RSP** always points to the **top** of the stack.
- **RBP** is the **base pointer**. It is used as a fixed reference point for the current function, so its local variables can be found at `[rbp - 8]`, `[rbp - 16]`, and so on, even while RSP moves.
- `call` automatically **pushes the return address** (the address of the next instruction) onto the stack.
- `ret` automatically **pops the return address** into the instruction pointer, so execution continues after the `call`.
- This program uses the stack to make a **stack frame**: a small block of space on the stack that belongs to the function `work`.

In this README, `RSP (start)` means the value RSP has when the program begins.

## What happens from start to end

### 1. Program starts
Execution begins at `_start`. RSP is at `RSP (start)`. Nothing from our program is on the stack yet.

### 2. `call work`
`call` does two things:
1. It pushes the **return address** onto the stack. This is the address of the next instruction, `mov rax, 60`. RSP goes down by 8 (`RSP (start) - 8`).
2. It jumps to `work`.

```
Top  ->  return address
```

### 3. `push rbp`
This saves the caller's RBP value so it can be restored later. RSP goes down by 8 (`RSP (start) - 16`).

```
Top  ->  saved RBP
         return address
```

### 4. `mov rbp, rsp`
RBP now copies RSP, so RBP points at the saved RBP. This is the **base of the stack frame** of `work`. RSP does not change (`RSP (start) - 16`).

### 5. `sub rsp, 16`
This **reserves 16 bytes** on the stack for local variables (two 8-byte values) by moving RSP down by 16 (`RSP (start) - 32`). Nothing is stored yet; the space is simply made available.

```
         saved RBP         <- RBP points here
         return address
Top  ->  [16 bytes reserved for local variables]
```

### 6. `mov qword [rbp - 8], 100` and `mov qword [rbp - 16], 200`
These store values into the reserved space, using RBP as the reference.
- `100` goes at `[rbp - 8]`, the first local variable.
- `200` goes at `[rbp - 16]`, the second local variable.

These are normal writes to memory, **not** `push` instructions, so RSP does not change (it stays at `RSP (start) - 32`).

```
Higher addresses
   return address
   saved RBP         <- RBP
   100               <- [rbp - 8]
   200               <- [rbp - 16]   <- RSP (top)
Lower addresses
```

### 7. `mov rax, [rbp - 8]` and `mov rbx, [rbp - 16]`
The values are **read** back from the stack into registers: `rax = 100` and `rbx = 200`. Reading does not remove them, and RSP does not change.

### 8. `nop`
Does nothing. It marks the point where the whole frame is set up: `rax = 100`, `rbx = 200`, RSP at `RSP (start) - 32`.

### 9. `mov rsp, rbp`
This **removes the local variable space** by moving RSP back up to where RBP points. RSP is now `RSP (start) - 16`. The two local values are no longer part of the stack, though they are still sitting in memory.

### 10. `pop rbp`
This takes the saved RBP off the top of the stack and puts it back in RBP, **restoring the caller's RBP**. RSP goes up by 8 (`RSP (start) - 8`).

```
Top  ->  return address
```

### 11. `ret`
`ret` pops the return address off the top of the stack and jumps to it. RSP goes up by 8 and is back to `RSP (start)`. Execution continues at `mov rax, 60` in `_start`.

### 12. Exit
`mov rax, 60`, `xor rdi, rdi` and `syscall` call the exit system call with code 0. The program ends.

## How RSP changed

| Instruction | RSP | What changed on the stack |
|---|---|---|
| Start | RSP (start) | Empty |
| `call work` | RSP (start) − 8 | Return address pushed |
| `push rbp` | RSP (start) − 16 | Old RBP saved |
| `mov rbp, rsp` | RSP (start) − 16 | No change to RSP. RBP now marks the frame |
| `sub rsp, 16` | RSP (start) − 32 | 16 bytes reserved for locals |
| `mov [rbp-8], 100` | RSP (start) − 32 | 100 stored. RSP unchanged |
| `mov [rbp-16], 200` | RSP (start) − 32 | 200 stored. RSP unchanged |
| `mov rax/rbx, [...]` | RSP (start) − 32 | Values read. RSP unchanged |
| `mov rsp, rbp` | RSP (start) − 16 | Locals released |
| `pop rbp` | RSP (start) − 8 | Old RBP restored |
| `ret` | RSP (start) | Return address popped. Stack back to start |

## Observations from running the program

I assembled it with `nasm -f elf64`, linked it with `ld`, ran it, and stepped through it in GDB using `stepi`, `info registers rsp rbp` and `x/4gx $rsp`.

- The program prints nothing and exits normally with code `0`.
- `call` lowered RSP by 8 without any `push` instruction, because it pushes the return address itself. `ret` raised RSP by 8 for the same reason.
- `sub rsp, 16` made room for two local variables in one step. It did not store anything.
- The stores to `[rbp - 8]` and `[rbp - 16]` did **not** change RSP. Writing into reserved stack space is different from pushing.
- Looking at memory in GDB after the stores showed the layout from top to bottom: return address, saved RBP, `100`, `200`. `200` was at the lowest address, where RSP was pointing.
- After `mov rax, [rbp - 8]` and `mov rbx, [rbp - 16]`, `rax` was `100` and `rbx` was `200`.
- `mov rsp, rbp` followed by `pop rbp` undid the frame setup, in the reverse order it was created.
- RSP finished at the same value it started with, so the stack was balanced and `ret` returned to the correct place.

## Conclusion

In this program, the stack is used to make a stack frame for the function `work`. `call` saves the return address, `push rbp` and `mov rbp, rsp` set up the frame, and `sub rsp, 16` reserves space for two local variables, which hold `100` and `200`. When the function finishes, `mov rsp, rbp` and `pop rbp` tear the frame down, and `ret` uses the saved return address to go back to `_start`. RSP moves down as the frame is built and back up as it is removed, so it returns to its starting value before the program exits.