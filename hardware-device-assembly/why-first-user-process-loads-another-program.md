# Why first user process loads another program?

## What happen?

The first user process from `userinit()` loads `initcode.S` program, which internally calls `exec` to run `init` program.

#### The flow:

`userinit` prepares the first process. It sets the program counter to instructions in `initcode.S`. When scheduler runs it, it will begin execution. `initcode.S` has a system call `exec` to run `init`, and starts from there.

## Why the process cannot call `init` directly?

#### My original Thought

I think `userinit` can definitely call `init.c`. We can get the hex dump of `init.c` binary, then make our array `initcode[]` to contain it. The reason MIT implementers did not do it because there are a lot of hex to copy! But, I might be wrong…

## The Truth

The instruction of the loaded program must be fitted into one page.

```c
// Load the user initcode into address 0 of pagetable,
// for the very first process.
// sz must be less than a page.
void
uvminit(pagetable_t pagetable, uchar *src, uint sz)
{
  char *mem;

  if(sz >= PGSIZE)
    panic(“inituvm: more than a page”);
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  mappages(pagetable, 0, PGSIZE, (uint64)mem, PTE_W|PTE_R|PTE_X|PTE_U);
  memmove(mem, src, sz);
}
```

`initcode.S` is built to a binary file.

```text
$U/initcode: $U/initcode.S
    $(CC) $(CFLAGS) -nostdinc -I. -Ikernel -c $U/initcode.S -o $U/initcode.o
    $(LD) $(LDFLAGS) -N -e start -Ttext 0 -o $U/initcode.out $U/initcode.o
```

The obj dump of the binary shows the first instruction is `auipc a0,0x0`:

```text
# exec(init, argv)
.globl start
start:
        la a0, init
   0:    00000517              auipc    a0,0x0
   4:    00050513              mv    a0,a0
        la a1, argv
   8:    00000597              auipc    a1,0x0
   c:    00058593              mv    a1,a1
        li a7, SYS_exec
  10:    489d                    li    a7,7
        ecall
  12:    00000073              ecall
```

The program is mapped to the process at virtual address\(VA\) 0. The program counter is set to VA 0 as well, which is the 1st instruction.

```c
 // prepare for the very first "return" from kernel to user.
 p->tf->epc = 0;      // user program counter
 p->tf->sp = PGSIZE;  // user stack pointer
```

When CPU scheduler runs this process, the PC points to first instruction at VA 0. Then PC moves to the next instruction \(PC + 4\).

## Understand `initcode` section in Makefile

```text
$U/initcode: $U/initcode.S
    $(CC) $(CFLAGS) -nostdinc -I. -Ikernel -c $U/initcode.S -o $U/initcode.o
    $(LD) $(LDFLAGS) -N -e start -Ttext 0 -o $U/initcode.out $U/initcode.o
    $(OBJCOPY) -S -O binary $U/initcode.out $U/initcode
    $(OBJDUMP) -S $U/initcode.o > $U/initcode.asm
```

### Use linker for ELF executable

In Makefile: `$(LD) $(LDFLAGS) -N -e start -Ttext 0 -o $U/initcode.out $U/initcode.o`

[ld\(1\): GNU linker - Linux man page](https://linux.die.net/man/1/ld) 

`ld -o  /lib/crt0.o hello.o -lc` This tells `ld` to produce a file called _output_ as the result of linking the file “/lib/crt0.o” with “hello.o” and the library “libc.a”, which will come from the standard search directories. \(See the discussion of the **-l** option below.\)

The linker cmd generates `initcode.out` from object file. The output file is:

```text
$ file initcode.out
initcode.out: ELF 64-bit LSB executable, UCB RISC-V, version 1 (SYSV), statically linked, with debug_info, not stripped
```

### Use `objcopy` to generate stripped binary

In Makefile: `$(OBJCOPY) -S -O binary $U/initcode.out $U/initcode`

#### Research

[objcopy \(GNU Binary Utilities\)](https://sourceware.org/binutils/docs/binutils/objcopy.html) `objcopy` can be used to generate a raw binary file by using an output target of ‘binary’ \(e.g., use -O binary\).

When `objcopy` generates a raw binary file, it will essentially **produce a memory dump of the contents of the input object file**. All symbols and relocation information will be discarded. The memory dump will start at the load address of the lowest section copied into the output file.

Use **-S** to remove sections containing debugging information.

#### Explain

The result data file \(`initcode`\) is a stripped version of `initcode.out` \(binary\). It is a memory dump of the contents.

Look at this assembly

```text
       la a0, init
   0:    00000517              auipc    a0,0x0
   4:    00050513              mv    a0,a0
        la a1, argv
   8:    00000597              auipc    a1,0x0
   c:    00058593              mv    a1,a1
        li a7, SYS_exec
  10:    489d                    li    a7,7
        ecall
  12:    00000073              ecall
```

Then look at the hex dump:

```text
$ od -t xC initcode
0000000    17  05  00  00  13  05  05  02  97  05  00  00  93  85  05  02
0000020    9d  48  73  00  00  00  89  48  73  00  00  00  ef  f0  bf  ff
0000040    2f  69  6e  69  74  00  00  01  20  00  00  00  00  00  00  00
0000060    00  00  00
0000063
```

Now tell me, what do you observe? The hex dump is the assembly instructions in little endian format! \(`00000517` becomes `17 05 00 00` …\)

Now, we understand why there is a hex dump array loaded to xv6 first user process!!!

**Ref**

[objcopy\(1\): copy/translate object files - Linux man page](https://linux.die.net/man/1/objcopy)

Makefile prints tool prefix.

```text
what:
    @echo “$(TOOLPREFIX)”
```

```text
riscv64-unknown-elf-objcopy -S -O binary _echo\ 2 echo_little
```

