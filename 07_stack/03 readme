# 03.asm – Stack Explanation

## Program

```nasm
section .text
    global _start

_start:
    mov rax, 10
    call calculate

after_call:
    ; RAX should now be 15
    nop

    mov rax, 60
    xor rdi, rdi
    syscall

calculate:
    add rax, 5
    ret
```

## Quick idea of how the stack works here

- The stack grows **downward**. Pushing makes RSP **smaller** and popping makes RSP **bigger**, by 8 bytes each time.
- **RSP** always points to the **top** of the stack.
- `call` automatically **pushes the return address** (the address of the instruction right after the `call`) onto the stack, then jumps to the function.
- `ret` automatically **pops that return address** off the top of the stack and jumps back to it.
- In this program, the **only** thing the stack is used for is storing the return address. There are no `push` or `pop` instructions and no local variables.

In this README, `RSP (start)` means the value RSP has when the program begins.

## What happens from start to end

### 1. Program starts
Execution begins at `_start`. RSP is at `RSP (start)`. Nothing from our program is on the stack yet.

### 2. `mov rax, 10`
The number `10` is loaded into `rax`. This only affects a register, so RSP does not change.

### 3. `call calculate`
`call` does two things:
1. It **pushes the return address** onto the stack. The return address is the address of the next instruction, which is the one labelled `after_call`. RSP goes down by 8 (`RSP (start) - 8`).
2. It **jumps** to `calculate`.

```
Top  ->  return address (the address of after_call)
```

The stack now remembers where to come back to.

### 4. `add rax, 5`
Inside `calculate`, `rax` changes from `10` to `15`. This is only a register operation, so the stack and RSP are **not** touched. The return address stays on the stack and waits.

### 5. `ret`
`ret` **pops the return address** from the top of the stack and jumps to it. RSP goes up by 8 and is back to `RSP (start)`.

```
(stack is empty again)
```

Execution continues at `after_call`, right after the original `call`.

### 6. `after_call: nop`
Does nothing. At this point `rax = 15` (as the comment in the code says) and RSP is back at `RSP (start)`.

### 7. Exit
`mov rax, 60`, `xor rdi, rdi` and `syscall` call the exit system call with code 0. The stack is not used here and the program ends.

## How RSP changed

| Instruction | RSP | Stack contents (top first) | rax |
|---|---|---|---|
| Start | RSP (start) | empty | 0 |
| `mov rax, 10` | RSP (start) | empty | 10 |
| `call calculate` | RSP (start) − 8 | return address | 10 |
| `add rax, 5` | RSP (start) − 8 | return address | 15 |
| `ret` | RSP (start) | empty | 15 |
| `nop` | RSP (start) | empty | 15 |

The stack only changed twice: once when `call` pushed the return address and once when `ret` popped it.

## Observations from running the program

I assembled it with `nasm -f elf64`, linked it with `ld`, ran it, and stepped through it in GDB using `stepi` and `info registers rsp rax rip`.

- The program prints nothing and exits normally with code `0`.
- `call` lowered RSP by 8 even though there is no `push` instruction. `call` pushes the return address by itself.
- Looking at the top of the stack in GDB (`x/1gx $rsp`) after the `call` showed the address of `after_call`. This confirms the return address is the instruction right after the `call`.
- `add rax, 5` did not change RSP. Register arithmetic does not use the stack.
- `ret` raised RSP by 8 and execution continued at `after_call`, which proves `ret` used the saved return address.
- `rax` was `15` after returning, so the result of the function was passed back in a register.
- RSP finished at the same value it started with, so the stack was balanced.

## Conclusion

This is the simplest use of the stack for a function call. `call` pushes the return address so the program knows where to come back to, the function `calculate` does its work in a register (`10 + 5 = 15`), and `ret` pops the return address and jumps back. RSP goes down by 8 on the `call` and up by 8 on the `ret`, ending at its starting value before the program exits.