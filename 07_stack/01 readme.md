# 01.asm – Stack Explanation

## Program

```nasm
section .text
    global _start

_start:
    mov rax, 100
    mov rbx, 200

    push rax
    push rbx
    nop

    pop rcx
    pop rdx
    nop

    mov rax, 60
    xor rdi, rdi
    syscall
```

## Quick idea of how the stack works

- The stack is temporary storage that works **LIFO** (Last In, First Out): the last value pushed is the first value popped.
- **RSP** points to the **top** of the stack.
- The stack **grows downward**, so `push` makes RSP **smaller** and `pop` makes RSP **bigger**.
- Each 64-bit push or pop moves RSP by **8 bytes**.

In this README, `RSP (start)` means the value RSP has when the program begins.

## What happens from start to end

### 1. Program starts
Execution begins at `_start`. RSP is at `RSP (start)`. Our program has put nothing on the stack yet.

### 2. `mov rax, 100` and `mov rbx, 200`
The numbers go into registers `rax` and `rbx`. The stack is not involved, so RSP does not change.

### 3. `push rax`
RSP goes down by 8 (`RSP (start) - 8`) and the value `100` is stored at the new top.

```
Top  ->  100
```

### 4. `push rbx`
RSP goes down by another 8 (`RSP (start) - 16`) and `200` is stored on top of `100`.

```
Top  ->  200
         100
```

`200` is now at the top because it was pushed last.

### 5. `nop`
Does nothing. It marks the point where the stack holds both values (`200` on top, `100` below), and RSP is `RSP (start) - 16`.

### 6. `pop rcx`
The value at the top of the stack, `200`, is copied into `rcx`. Then RSP goes up by 8 (`RSP (start) - 8`).

```
Top  ->  100
```

`200` came out first because it was on top. This is LIFO.

### 7. `pop rdx`
The next value, `100`, is copied into `rdx`. RSP goes up by 8 and is back to `RSP (start)`.

```
(stack is empty again)
```

### 8. `nop`
Stack is back to how it was at the start. Registers now hold `rcx = 200` and `rdx = 100`.

### 9. Exit
`mov rax, 60`, `xor rdi, rdi` and `syscall` call the exit system call with code 0. The stack is not used here and the program ends.

## How RSP changed

| Instruction | RSP | Stack contents (top first) |
|---|---|---|
| Start | RSP (start) | empty |
| `push rax` | RSP (start) − 8 | 100 |
| `push rbx` | RSP (start) − 16 | 200, 100 |
| `pop rcx` | RSP (start) − 8 | 100 |
| `pop rdx` | RSP (start) | empty |

Two pushes lowered RSP by 16 bytes (2 × 8). Two pops raised it by 16 bytes, so it ended where it began.

## Observations from running the program

I assembled it with `nasm -f elf64`, linked it with `ld`, ran it, and stepped through it in GDB using `stepi` and `info registers rsp`.

- The program prints nothing and exits normally with code `0`.
- RSP dropped by 8 after each `push` and rose by 8 after each `pop`, which shows the stack grows downward.
- The values came out in **reverse order**: `rcx` received `200` and `rdx` received `100`, even though `100` was pushed first. This is LIFO.
- `rax` and `rbx` still held `100` and `200` after the pushes, because `push` copies the value and does not remove it from the register.
- RSP finished at the same value it started with, so the number of pushes matched the number of pops.
- After the pops, the old values were still in memory below RSP. `pop` only moves RSP; it does not erase the data.

## Conclusion

The program pushes `100` and then `200` onto the stack. Because the stack is LIFO, `200` (the top) is popped first into `rcx`, and then `100` is popped into `rdx`. RSP moves down by 8 on each push and up by 8 on each pop, and returns to its starting value before the program exits.