+++
title = "RTOS Context Switching"
draft = false
tags = ["RTOS", "ARM Cortex-M4", "C", "ARM Assembly"]
ShowReadingTime = true
date = 2026-03-18
+++

This post is going to be a deep-dive into how context switching
works in my RTOS learning project.

90% of the code is going to be in
ARM Assembly, so before we touch on anything else, below is a link to a
"cheat sheet" of the ARM Assembly operations used in the context switching logic.
It is by no means a comprehensive list, but it will be sufficient for understanding
the assembly that is used in this context.

I am assuming you are familiar with C,
so I will provide some very rough analog of equivalent logic in C for each operation.

[ARM Assembly Reference](/pages/arm-assembly-reference/)

---

## The Hardware Context

Before we can talk about switching tasks, we need to understand how the
Cortex-M4 thinks about execution. The CPU operates in one of two modes at
any given time:

- **Handler mode** — where exceptions and interrupts run. Always uses the
  Main Stack Pointer (MSP).
- **Thread mode** — where normal application code runs. In my RTOS, this
  is configured to use the Process Stack Pointer (PSP).

Every task in the RTOS gets its own stack. The PSP points to the top of
whichever task is currently running. The MSP is reserved for the kernel and
interrupt handlers. This separation is what makes context switching possible;
swapping tasks is, at its core, just swapping the PSP.

---

## What the CPU Saves For Free (And What It Doesn't)

When an interrupt fires, the Cortex-M4 hardware automatically pushes eight
registers onto the current task's PSP before entering the handler:

```
xPSR, PC, LR, R12, R3, R2, R1, R0
```

This is called the **exception frame**. The CPU does this so that when the
interrupt returns, it can restore execution exactly where it left off.

The problem is that the CPU only saves half the picture. There are sixteen
core registers total. The remaining eight, `R4` through `R11`, are
callee-saved registers. The C ABI says any function that uses them is
responsible for saving and restoring them. In our case, the RTOS is that
function. We have to save them ourselves.

So a full context for one task looks like this:

```
[ R4 - R11  ]  <-- software-saved by PendSV_Handler
[ EXC_RETURN]  <-- software-saved (the LR value at exception entry)
[ R0 - R3   ]  <-- hardware-saved automatically
[ R12       ]  <-- hardware-saved automatically
[ LR        ]  <-- hardware-saved automatically
[ PC        ]  <-- hardware-saved automatically
[ xPSR      ]  <-- hardware-saved automatically
```

The hardware frame sits at the bottom. Our software frame sits on top of it.
Together they represent the complete CPU state of a paused task.

---

## PendSV (The Deferred Switch)

This RTOS uses the **PendSV** exception to perform context switches. PendSV is
a hardware exception, it lives in the vector table and is dispatched by the
CPU like any other interrupt. But it's unique: nothing in the hardware fires
it automatically. You trigger it yourself by writing to a register.

When a hardware interrupt (UART, I2C, a timer) decides that a higher-priority
task should run, it doesn't context switch immediately. It just sets the PendSV
pending bit in the System Control Block's Interrupt Control and State Register
(SCB_ICSR):

```asm
trigger_context_switch:
    ldr     r0, =SCB_ICSR       // address: 0xE000ED04
    ldr     r1, =ICSR_PENDSVSET // value:   0x10000000 (bit 28)
    str     r1, [r0]
    dsb
    isb
    bx      lr
```

The `dsb` and `isb` barrier instructions ensure the write completes and the
pipeline is flushed before returning. Once all higher-priority exceptions have
finished, PendSV fires and the actual switch happens.

This is why PendSV is set to the lowest possible hardware priority (0xFF) in
the reset handler:

```asm
ldr r0, =0xE000ED22   // PendSV priority register
mov r1, #0xFF         // lowest priority
strb r1, [r0]
```

The priority hierarchy looks like this:

```
NMI / HardFault              <- can never be masked
Peripheral interrupts (NVIC) <- preempt everything below
─────────────────────────────
PendSV  (0xFF)               <- context switch, runs last
SysTick (0xFF)               <- tick timer, runs last
─────────────────────────────
RTOS tasks                   <- software only, CPU has no idea
```

One important detail: PendSV has a single pending bit. If two interrupts both
call `trigger_context_switch` before PendSV gets a chance to run, the second
write is a no-op, the bit is already set. PendSV fires once, the scheduler
picks the highest-priority ready task, and the switch happens. No queue needed.

---

## Initializing a New Task

When a task is created for the first
time, it has never run. It has no saved context. But `PendSV_Handler` is going
to try to restore a context from its stack unconditionally.

The solution is to lie to it. When a task is created, we pre-populate its
stack to look exactly as if the task was already running and got interrupted.
This is called the **fake stack frame**.

In `task.c`, `task_init_stack()` builds the stack from top-down:

```
High address (top of stack)
┌─────────────┐
│  xPSR       │  0x01000000  <- Thumb bit must be set
│  PC         │  task_entry  <- where the task starts
│  LR         │  0x00000000  <- tasks shouldn't return
│  R12        │  0x00000000
│  R3         │  0x00000000
│  R2         │  0x00000000
│  R1         │  0x00000000
│  R0         │  task_param  <- argument passed to task
├─────────────┤  <- hardware frame ends here
│  LR (EXC)  │  0xFFFFFFFD  <- EXC_RETURN: return to Thread/PSP
│  R11 - R4  │  0x00000000  <- all zeroed
└─────────────┘
Low address    <- tcb->stack_pointer set here
```

The `0xFFFFFFFD` EXC_RETURN value is a special sentinel the CPU recognizes
when it sees it in LR during an exception return. It tells the hardware:
"return to Thread mode and pop the hardware frame from the PSP." When
`PendSV_Handler` runs `bx lr` with this value, the CPU does exactly that;
it pops `R0`–`R3`, `R12`, `LR`, `PC`, and `xPSR` off the task's stack and
resumes execution at the task's entry point.

The `0x01000000` in xPSR sets the Thumb bit. On Cortex-M4, all instructions
are Thumb. If this bit isn't set when the CPU tries to execute the task,
you get an immediate UsageFault.

---

## Starting the First Task

We can't use `PendSV_Handler` to start the very first task because we're
currently running on the MSP (the system stack from `Reset_Handler`), and
there is no "current task" to save. We need a one-time bootstrap.

`start_first_task()` in `context_switch.s` takes the first task's stack
pointer in `R0` and manually unwinds it:

```asm
start_first_task:
    msr     psp, r0         // point PSP at the first task's stack

    mrs     r0, control
    orr     r0, r0, #2      // set bit 1: use PSP in Thread mode
    msr     control, r0
    isb                     // flush pipeline after CONTROL change

    cpsie   i               // enable interrupts

    pop     {r4-r11, lr}    // unwind software frame
    pop     {r0-r3, r12}    // unwind hardware frame (partially)
    pop     {r1}            // discard saved LR
    pop     {pc}            // jump to task entry point
```

The `pop {pc}` is the trigger. Popping a value directly into `PC` causes an
immediate branch to that address — in this case, the task's entry function.
From this point on, the CPU is in Thread mode using the PSP, and the RTOS
is running.

The `cpsie i` deserves a mention. When `st-flash` resets the board, it can
leave `PRIMASK` set (interrupts globally disabled). If we don't explicitly
clear it here, SysTick and PendSV never fire and the scheduler never runs.

---

## The Core Switcher: PendSV_Handler

This is the heart of the RTOS. Every context switch passes through here.
When PendSV fires, the CPU has already saved the hardware frame
(`R0`–`R3`, `R12`, `LR`, `PC`, `xPSR`) onto the current task's PSP.
Our job is to save the rest, pick the next task, and restore its state.

### Phase 1: Save the current context

```asm
PendSV_Handler:
    cpsid   i                   // disable interrupts

    ldr     r2, =current_task
    ldr     r1, [r2]            // r1 = current_task pointer
    cbz     r1, switch_logic    // if NULL, skip save (first switch)

    mrs     r0, psp             // read current PSP
    stmdb   r0!, {r4-r11, lr}   // push R4-R11 + EXC_RETURN onto stack
    str     r0, [r1]            // save updated SP into TCB
```

`stmdb`  (Store Multiple, Decrement Before) pushes each register by
first decrementing the address, then writing. The `!` writes the final
address back into `R0`. After this, `R0` points to the new top of the
task's stack, which is immediately saved into the TCB's `stack_pointer`
field with `str r0, [r1]`.

### Phase 2: Pick the next task

```asm
switch_logic:
    bl      scheduler_switch_context
    // r0 now holds the new current_task pointer
```

We branch out to C. `scheduler_switch_context()` reads the DWT cycle
counter for profiling, calls `scheduler_get_next_task()` to pull the
highest-priority ready task off the ready queues in O(1), updates the
global `current_task`, and returns the new task pointer in `R0`.

### Phase 3: Restore the new context

```asm
    ldr     r0, [r0]            // r0 = next_task->stack_pointer

    ldmia   r0!, {r4-r11, lr}   // pop R4-R11 + EXC_RETURN off stack
    msr     psp, r0             // update PSP to new task's stack

    cpsie   i                   // re-enable interrupts
    bx      lr                  // exception return via EXC_RETURN
```

`ldmia` (Load Multiple, Increment After) is the mirror of `stmdb`.
It reads registers from memory, incrementing the address after each load.
After this, `R4`–`R11` hold the new task's callee-saved registers, `LR`
holds `0xFFFFFFFD`, and the PSP points just above the hardware frame on
the new task's stack.

The final `bx lr` is the magic. Because `LR` contains `0xFFFFFFFD`, the
CPU recognizes this as an exception return, looks at the PSP, and pops the
hardware frame (`R0`–`R3`, `R12`, `LR`, `PC`, `xPSR`) off the new
task's stack. Execution resumes at the new task's `PC`, with its complete
register state restored. From the task's perspective, nothing happened.

---

## The Tick (SysTick_Handler)

`SysTick` fires at whatever frequency `systick_init()` configures, in
this RTOS, 1ms. Its handler is deliberately minimal:

```asm
SysTick_Handler:
    push    {r7, lr} // r7 pads to maintain 8-byte stack alignment
    bl      scheduler_tick
    pop     {r7, lr}
    bx      lr
```

`scheduler_tick()` increments `tick_now`, walks the timing wheel bucket for the current tick
to wake any tasks whose timeout has expired, then unconditionally calls trigger_context_switch()
to set the `PendSV` bit. `SysTick` itself never context switches, it just nudges `PendSV` and lets it
do the work once `SysTick` exits.

---

## Putting It All Together

A full context switch triggered by SysTick looks like this end-to-end:

```
1ms elapses
  → SysTick fires (priority 0xFF)
  → scheduler_tick() increments tick count
  → a delayed task's timeout expires
  → trigger_context_switch() sets PendSV bit
  → SysTick exits

PendSV fires (priority 0xFF, runs now that SysTick has exited)
  → disable interrupts
  → read PSP of current task
  → stmdb: push R4-R11, EXC_RETURN onto current task's stack
  → save updated SP into current TCB
  → bl scheduler_switch_context (pick next task)
  → ldr: load next task's SP
  → ldmia: pop R4-R11, EXC_RETURN off next task's stack
  → update PSP
  → re-enable interrupts
  → bx lr (EXC_RETURN = 0xFFFFFFFD)

Hardware exception return
  → CPU sees 0xFFFFFFFD in LR
  → pops R0-R3, R12, LR, PC, xPSR from PSP
  → jumps to next task's PC
  → next task resumes, unaware anything happened
```

The entire switch, from SysTick firing to the next task running, happens
in a handful of microseconds (~4μs!), entirely in hardware-enforced priority order,
with no task ever aware it was paused.

---

That concludes the walkthrough of how context switching is implemented for my custom RTOS.

I abstracted some of the C logic to focus more on the context switching, so if you want to check out the entire codebase, feel free to do so [here](https://github.com/CldStlkr/morph-rt)
