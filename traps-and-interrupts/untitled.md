---
description: 'What happens if the shell printing a prompt with write(2, “$ “, 2)?'
---

# How system calls get into/out of the kernel

## Key Concepts

### The overall trajectory

```c
  write()
    trampoline / uservec
      usertrap() in trap.c
        syscall() in syscall.c
          sys_write() in sysfile.c
      usertrapret()
    trampoline / userret
  write()
```

#### what’s the state of the machine at this point?

user address space, kernel address space trampoline at top of user address space trapframe just under the trampoline kernel stack for shell in the kernel

#### trapframe content, set up in advance by kernel

\[on board\] see proc.h kernel page table kernel stack pointer address of usertrap\(\) function in kernel space for saved user pc space for 32 saved user registers

#### special RISC-V registers \(many more, here are relevant ones\)

Chapter 10 in The RISC-V Reader \[on board\] stvec — ecall jumps here in kernel; address of trampoline sepc — ecall saves user pc here scause — ecall sets to 8 to indicate a system call sscratch — address of trapframe satp — current page table

#### C function calling convention on RISC-V

**i.e. how function calls use the 32 RISC-V registers** important since system calls start as C function calls e.g. shell’s call to write\(\) a0..a7 — arguments ra — return address a0 — return value

Questions: 1. What does trampoline do? \[\[What is trampoline\]\] 2. What does trapframe do? \[\[What is trapframe\]\] 3. What does RISV-V registers do in details? 4. Why cannot use functional all for system call?

**Can’t use function call for system call!** Due to the need for isolation. Isolation looms over much of O/S design. See \[\[OS isolation concepts\]\]

## The flow

### Phase 1\(set up\)

1. When executing user code, register stvec is set to `uservec`.
2. Traps from user space start from uservec.
3. Another register sscratch points to trapframe.

**When it needs to force a trap, the RISC-V hardware does the following for all trap types \(other than timer interrupts\):** 1. If the trap is a device interrupt, and the sstatus SIE bit is clear, don’t do any of the following. 2. Disable interrupts by clearing SIE. 3. Copy the pc to sepc. 4. Save the current mode \(user or supervisor\) in the SPP bit in sstatus. 5. Set scause to reflect the interrupt’s cause. 6. Set the mode to supervisor. 7. Copy stvec to the pc. 8. 8. Start executing at the new pc.

### Phase 2\(trampoline to kernel\)

1. User program executes a system call, called `ecall`
2. Jump to $stvec \(i.e. set $pc to $stvec\)
3. Save $pc in $sepc
4. Change mode from user to supervisor \(we cannot see this in code\).
5. Start running code from `uservec`.
6. Save user registers in TRAPFRAME\(aka tf\) page.
7. Restore kernel stack pointer from tf page.
8. Hold current hartid\(???\)
9. Load the address of usertrap\(\), the kernel C code to handler user trap.
10. Restore kernel page table from tf page.
11. Jump to usertrap\(\).

**Note:** 1. ecall lets a user program switch to privileged supervisor mode but it does let the user program control what instructions are executed in that mode. 2. ecall always jumps to $stvec, which only can be written in supervisor mode \(not by user programs\), and which the kernel carefully set up to a known entry point \(trampoline\). 3. After switching kernel address space, save all registers, point $sp to top of kernel stack, then kernel can run C functions. 4. 32 user registers are stored in trapframe. Kernel set a special register to the address of the trapframe, so we know where to find the trapframe page.

### Phase 3 \(Handle system call trap\)

We jump to usertrap\(\) kernel C function to handle an interrupt, exception, or system call from user space.

### Phase 4 \(Prepare to return to user space\)

First step is the call to usertrapret \(kernel/trap.c:90\). This function sets up the RISC-V control registers to prepare for a future trap from user space. Involving change stvec to uservec. Prepare the trapframe fields. Set spec to previously saved user PC.

### Phase 5 \(Jump to user page table\)

```c
  // tell trampoline.S the user page table to switch to.*
  uint64 satp = MAKE_SATP(p->pagetable);

  // jump to trampoline.S at the top of memory, which*
  // switches to the user page table, restores user registers,
  // and switches to user mode with sret.
  uint64 fn = TRAMPOLINE + (userret - trampoline);
  ((void (*)(uint64,uint64))fn)(TRAPFRAME, satp);
```

It jumps to `userret`, and switch to user page table. satp is passed in as a param.

### Phase 6 \(Return to user mode and user PC\)

1. Switch to user page table
2. Restore registers.
3. Return to user mode and user pc.

