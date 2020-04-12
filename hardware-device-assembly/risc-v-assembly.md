# RISC-V assembly

## In RISC-V world, what does `JALR` instruction do? What does `ra` register do? What does `ret` assembly do?

`ra` is ABI name for register `x1`. It is to store return address.

`ret` : Return from subroutine. The actual instructions are:

```text
jalr x0, x1, 0
auipc x6, offset[31:12]
```

#### Code Example

```text
void g(int x) {
  printf("%d", x);
}

void main(void) {
  g(3);
  exit(0);
}
```

**Assembly version**

```text
void g(int x) {
   0: 1141                 addi  sp,sp,-16
   2: e406                 sd ra,8(sp)
   4: e022                 sd s0,0(sp)
   6: 0800                 addi  s0,sp,16
   8: 85aa                 mv a1,a0
  printf(“%d”, x)*;*
   a: 00000517             auipc a0,0x0
   e: 7ce50513             addi  a0,a0,1998*# 7d8 <malloc+0xe6>*
  12: 00000097             auipc ra,0x0
  16: 622080e7             jalr  1570(ra)*# 634 <printf>*
}
  1a: 60a2                 ld ra,8(sp)
  1c: 6402                 ld s0,0(sp)
  1e: 0141                 addi  sp,sp,16
  20: 8082                 ret

0000000000000022 <main>:

void main(void) {
  22: 1141                 addi  sp,sp,-16
  24: e406                 sd ra,8(sp)
  26: e022                 sd s0,0(sp)
  28: 0800                 addi  s0,sp,16
  g(3)*;*
  2a: 450d                 li a0,3
  2c: 00000097             auipc ra,0x0
  30: fd4080e7             jalr  -44(ra)*# 0 <g>*
  exit(0)*;*
  34: 4501                 li a0,0
  36: 00000097             auipc ra,0x0
  3a: 27e080e7             jalr  638(ra)*# 2b4 <exit>*
```

#### Explain

`main` uses `jalr` to call g\(\).

**What does jar do?**

RISC-V’s subroutine call jal \(jump and link\) places its return address in a register. This is faster in many computer designs, because it saves a memory access compared to systems that push a return address directly on a stack in memory.

In our example, the return address is saved in `ra`. Then PC jumps to g\(\) and continue executing. In beginning of g\(\), you can see it saves return address to stack: `sd ra,8(sp)` At end of g\(\), it retrieves `ra`back: `ld ra,8(sp)`

**What is return address?**

Assembly snippet

```text
  30: fd4080e7             jalr  -44(ra)*# 0 <g>*
  exit(0)*;*
  34: 4501                 li a0,0
```

Gdb breakpoints and exam content in `ra`

```text
Thread 3 hit Breakpoint 1, g (x=x@entry=3) at user/call.c:6
6    void g(int x) {
(gdb) info reg
ra             0x34    0x34 <main+18>
```

Clearly, the return address is the address of the next instruction after JAL, which is **0x34**.

**We found the proof in RISCV-SPEC** JAL stores the address of the instruction following the jump \(pc+4\) into register rd.

So before we calling function, we set return address first.

**what does ret do?**

Let’s see spec first: 

![](../.gitbook/assets/image%20%2844%29.png)

`jalr` saves return address to first param. This part we are clear. It then sets PC to rs1 + offset. So PC will continue from the new address. Now, let’s look at our ret instruction’s meaning

```text
jalr x0, x1, 0
auipc x6, offset[31:12]
```

Save return address to x0, which does nothing in our code example. It then set PC to `ra` + 0, which means let program counter continues from the return address we saved before!

#### Summary

When your program is calling a function, it prepares the return address first before jump to the function. So that when the function about to return, it knows where it should return to. `jalr` saves return address for current PC+4, jump to the calling function. `ra` contains the return address.

### When function returns, it calls `ret`, which set its program counter to `ra`. In this way, function is returned correctly and continue on the next line of code.

## RISC-V calling convention

![](../.gitbook/assets/image%20%2811%29.png)

### How to design Virtual Memory Area?

**Proposal 1**

Start from max VA - trampoline - trap frame. Every mmap, find current max end VA we use, deduct size, and round down to find start address. Save start addr, and end address.

Keep track of current max end virtual address. So every mmap knows where to start.

In munmap, we need to reset current max end if needed.

For searching, it is bad. Since it requires loop the entire list to find coverage.

### How unix pipe works

#### Initialization

The system call pipe creates a pipe object, and two files. The two files link to the pipe, one for write, one for read. The system calls write back the FD of two files to user program.

#### Runtime

Shell calls fork to run write, and another fork to run read. In this way, 2 different process are running write and read work, so they can wait and wakeup each other if needed. The detail implementation of how pipe write and read work is in pipe.c. But the key take-away is since they are handled by different process, so they can sleep on chan, and got waken up by scheduler.

#### Design

A file can be a pipe type. A pipe is:

```text
struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;     // number of bytes read*
  uint nwrite;    // number of bytes written*
  int readopen;   // read fd is still open*
  int writeopen;  // write fd is still open*
};
```

Two file write and read from the same pipe’s data array.

#### Brainstorming

**broadcast feature**

We could implement a broadcast system call similar to pipe. A write could be broadcast to multiple listeners. Once the writer is done, wakeup all sleeping processes, and let them read, and do something. Each listener updates how much they have read, last read position etc. How to run this new call?

```text
int fds[5]
broadcast(fds)
write(fds[0]
…
read(fds[1])
…
read(fds[2])
…
```

**IPC**

Can we make a ‘pipe' talking to each other? We might need 2 data arrary in struct IPC\_PIPE. One for fd1 write, one for fd2 write. Each pipe type file needs to remember it reads from which data content, and write to which one.

## What stores in register `ra` when context switching?

### Assembly Flow

Process calls `yield` to give up CPU, and let CPU scheduler to run. It calls the context switch assembly to save its registers, for later restoring purpose.

The assembly for calling the big `swtch` is at `800020b6`:

![](../.gitbook/assets/image%20%2836%29.png)

`switch` is at:

```text
0000000080002618 <swtch>:
    80002618:    00153023              sd    ra,0(a0)
    8000261c:    00253423              sd    sp,8(a0)
    80002620:    e900               sd    s0,16(a0)
    80002622:    ed04               sd    s1,24(a0)
    80002624:    03253023              sd    s2,32(a0)
    80002628:    03353423              sd    s3,40(a0)
    8000262c:    03453823              sd    s4,48(a0)
    80002630:    03553c23              sd    s5,56(a0)
    80002634:    05653023              sd    s6,64(a0)
…
```

### How it set `ra` to `pc+4`

Key code is here:

```text
800020ae:      addi    a0,s1,104
800020b2:   auipc    ra,0x0
800020b6:   jalr    1382(ra) # 80002618 <swtch>
800020ba:      mv    a5,tp
```

`auipc ra,0x0` prepares `ra` for the jump. `ra` is set to `800020b2`.

`jalr 1382(ra)` adds 0x1382 to `800020b2`, which is jumping to the `swtch` context swtich assembly \(at `80002618`\).

After the jump, at this point, `ra` is set to PC + 4, which is `800020ba`.

### See GDB debug references

![](../.gitbook/assets/image%20%2812%29.png)

`Stepi` shows the next instruction to execute.

### Summary

`Swtch` does not save the program counter. Instead, `swtch` saves the `ra` register, which holds the return address from which `swtch` was called \(PC + 4\).

