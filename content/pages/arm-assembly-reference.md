+++
title = 'ARM Assembly Reference'
date = 2026-03-15
draft = false
description = 'A quick-reference cheat sheet for ARM Cortex-M4 assembly instructions used in RTOS context switching.'
layout = 'single'
+++

A reference for the ARM Cortex-M4 assembly instructions used throughout the
Morph-RT context switching implementation. Not exhaustive, scoped to what
actually appears in `context_switch.s` and `startup.s`.

| Op | Example | What it does | C analog |
|---|---|---|---|
| `ldr` | `ldr r0, =SCB_ICSR` | Load a value or address into a register | `uint32_t r0 = SCB_ICSR;` |
| `str` | `str r1, [r0]` | Store a register's value to a memory address | `*r0 = r1;` |
| `mov` | `mov r2, #0` | Copy a value directly into a register | `r2 = 0;` |
| `orr` | `orr r0, r0, #2` | Logical OR. Set specific bits without touching others | `r0 = r0 \| 2;` |
| `bl` | `bl scheduler_switch_context` | Branch with link. Call a function, save return address in `lr` | `scheduler_switch_context();` |
| `bx` | `bx lr` | Branch to address in a register. Used to return from a function or exception | `return;` |
| `cbz` | `cbz r1, switch_logic` | Compare and branch if zero. Jump if register is `NULL` | `if (r1 == 0) goto switch_logic;` |
| `mrs` | `mrs r0, psp` | Read a special-purpose CPU register into a general register | `r0 = PSP;` |
| `msr` | `msr psp, r0` | Write a general register into a special-purpose CPU register | `PSP = r0;` |
| `stmdb` | `stmdb r0!, {r4-r11, lr}` | Store multiple registers to memory, decrementing address before each write (push onto descending stack). `!` writes the final address back to `r0` | `*(--r0) = r4; ... *(--r0) = lr;` |
| `ldmia` | `ldmia r0!, {r4-r11, lr}` | Load multiple registers from memory, incrementing address after each read (pop from stack). `!` writes the final address back to `r0` | `r4 = *(r0++); ... lr = *(r0++);` |
| `push` | `push {r7, lr}` | Push registers onto the current stack, decrement SP | `stack[--sp] = r7; stack[--sp] = lr;` |
| `pop` | `pop {r4-r11, lr}` | Pop registers off the current stack, increment SP | `lr = stack[sp++]; ... r4 = stack[sp++];` |
| `cpsid` | `cpsid i` | Disable interrupts globally (sets PRIMASK) | `__disable_irq();` |
| `cpsie` | `cpsie i` | Enable interrupts globally (clears PRIMASK) | `__enable_irq();` |
| `dsb` | `dsb` | Data Synchronization Barrier. Ensure all memory writes complete before continuing | `__DSB();` |
| `isb` | `isb` | Instruction Synchronization Barrier. Flush the pipeline, force re-fetch of instructions | `__ISB();` |
| `udiv` | `udiv r2, r1, r0` | Unsigned integer divide. `r2 = r1 / r0` | `r2 = r1 / r0;` |
| `sub` | `sub r2, r2, #1` | Subtract. `r2 = r2 - 1` | `r2 = r2 - 1;` |
