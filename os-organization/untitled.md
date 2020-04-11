# OS Organization

OS must arrange for isolation between the processes. If one process has a bug and fails, it shouldn’t affect others that don’t depend on the failed process.

An operating system must fulfill three requirements: **multiplexing, isolation, and interaction.**

Abstracting physical resources.

### Three modes that CPU supports

CPUs provide hardware support for strong isolation. For example, RISC-V has three modes in which the CPU can execute instructions: machine mode, supervisor mode, and user mode.

Instructions executing in machine mode have full privilege; a CPU starts in machine mode. Machine mode is mostly intended for configuring a computer.

In supervisor mode the CPU is allowed to execute privileged instructions: for example, enabling and disabling interrupts, reading and writing the register that holds the address of a page table, etc.

An application can execute only user-mode instructions.

### App wants to invoke a kernel function \(system call\)

An application that wants to invoke a kernel function \(e.g., the read system call in xv6\) must transition to the kernel. CPUs provide a special instruction that switches the CPU from user mode to supervisor mode and enters the kernel at an entry point specified by the kernel. \(RISC-V provides the `ecall` instruction for this purpose.\)

Kernel control the entry point for transitions to supervisor mode for security reasons.

### Process Overview

The unit of isolation in xv6 is a _process_. 

![](../.gitbook/assets/image%20%285%29.png)

#### Layout of a user process

![](../.gitbook/assets/image%20%2811%29.png)

#### Each process has its own separate page table.

Xv6 runs on RISC-V with 39 bits for virtual addresses, but uses only 38 bits. Thus, the maximum address is 2 38 − 1 = 0x3fffffffff, which is MAXVA \(kernel/riscv.h:349\).

#### Lifetime

Each process has a thread of execution \(or thread for short\) that executes the process’s instructions. A thread can be suspended and later resumed. To switch transparently between processes, the kernel suspends the currently running thread and resumes another process’s thread \(context switch\).

### Start xv6 and run the first process

#### Kernel start

When the RISC-V computer powers on, it initializes itself and runs a boot loader which is stored in read-only memory. The boot loader loads the xv6 kernel into memory. Then, in machine mode, the CPU executes xv6 starting at \_entry \(kernel/entry.S:12\). Xv6 starts with the RISC-V paging hardware disabled: virtual addresses map directly to physical addresses.

1. The boot loader loads the xv6 kernel into memory at physical address 0x80000000, because the address range 0x0:0x80000000 contains I/O devices.
2. The instructions at \_entry set up a stack so that xv6 can run C code. The code at \_entry loads the stack pointer register sp with the address `stack0+4096`, the top of the stack, because the stack on **RISC-V grows down**.
3. Now xv6 has a stack, and able to run C code.
4. \_entry calls into C code at start \(kernel/start.c:21\).
5. Perform some configurations that is only allowed in machine mode.
6. Set Program Counter to `main`.
7. Disable paging by set `satp` to 0.
8. Enable clock interrupts.
9. Switch to supervisor mode and jump to `main()`.

#### First process

In `main`, after initializes several devices and subsystems, it creates the first process by calling `userinit`.

```c
void
userinit(void)
{
  struct proc *p;

  p = allocproc();
  initproc = p;

  // allocate one user page and copy init’s instructions
  // and data into it.
  uvminit(p->pagetable, initcode, sizeof(initcode));
  p->sz = PGSIZE;

  // prepare for the very first “return” from kernel to user.
  p->tf->epc = 0;      // user program counter
  p->tf->sp = PGSIZE;  // user stack pointer

  safestrcpy(p->name, “initcode”, sizeof(p->name));
  p->cwd = namei(“/“);

  p->state = RUNNABLE;

  release(&p->lock);
}
```

The 1st process executes a small program: `initcode`. In code, we have

```c
// a user program that calls exec(“/init”)
// od -t xC initcode
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x05, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x05, 0x02,
  0x9d, 0x48, 0x73, 0x00, 0x00, 0x00, 0x89, 0x48,
  0x73, 0x00, 0x00, 0x00, 0xef, 0xf0, 0xbf, 0xff,
  0x2f, 0x69, 0x6e, 0x69, 0x74, 0x00, 0x00, 0x01,
  0x20, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00
};
```

The above is the [octal](https://en.wikipedia.org/wiki/Octal) data format of the binary built from `initcode.S`.

`initcode.S`:

```c
# Initial process execs /init.
# This code runs in user space.

#include “syscall.h”

# exec(init, argv)
.globl start
start:
        la a0, init
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = “/init\0”;
init:
  .string “/init\0”

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long init
  .long 0
```

The above `initcode` program re-enters kernel by invoking `exec` system call to run a new program `init`.

The `init`:

```c
int
main(void)
{
  int pid, wpid;

  if(open(“console”, O_RDWR) < 0){
    mknod(“console”, 1, 1);
    open(“console”, O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf(“init: starting sh\n”);
    pid = fork();
    if(pid < 0){
      printf(“init: fork failed\n”);
      exit(1);
    }
    if(pid == 0){
      exec(“sh”, argv);
      printf(“init: exec sh failed\n”);
      exit(1);
    }
    while((wpid=wait(0)) >= 0 && wpid != pid){
      //printf(“zombie!\n”);
    }
  }
}
```

This program creates file descriptors 0, 1, 2. Start a console shell. Wait for shell exists, and repeat.

**Now the system is up!**

