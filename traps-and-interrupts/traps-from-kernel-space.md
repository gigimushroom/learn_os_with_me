# Traps from kernel space

### Two kinds of traps can occur from kernel:

1. exceptions
2. device interrupts

## What happened during trap?

1. Save all registers
2. Call kerneltrap\(\) trap handler
3. Restore registers
4. Return by calling  `sret`, which copies `sepc` to program counter and resumes the interrupted kernel code.

## Implementation Details

Kernel points `stvec` register to `kernelvec` assmebly code. `kernelvec` save registers on the same interrupted kernel thread, since all registers belong to that thread.

```text
kernelvec:
        // make room to save registers.
        addi sp, sp, -256

        // save the registers.
        sd ra, 0(sp)
        sd sp, 8(sp)
        sd gp, 16(sp)
        sd tp, 24(sp)
        sd t0, 32(sp)
        sd t1, 40(sp)
        sd t2, 48(sp)
        sd s0, 56(sp)
        sd s1, 64(sp)
        sd a0, 72(sp)
        sd a1, 80(sp)
        sd a2, 88(sp)
        sd a3, 96(sp)
        sd a4, 104(sp)
        sd a5, 112(sp)
        sd a6, 120(sp)
        sd a7, 128(sp)
        sd s2, 136(sp)
        sd s3, 144(sp)
        sd s4, 152(sp)
        sd s5, 160(sp)
        sd s6, 168(sp)
        sd s7, 176(sp)
        sd s8, 184(sp)
        sd s9, 192(sp)
        sd s10, 200(sp)
        sd s11, 208(sp)
        sd t3, 216(sp)
        sd t4, 224(sp)
        sd t5, 232(sp)
        sd t6, 240(sp)

    // call the C trap handler in trap.c
        call kerneltrap

        // restore registers.
        ld ra, 0(sp)
        ld sp, 8(sp)
        ld gp, 16(sp)
        // not this, in case we moved CPUs: ld tp, 24(sp)
        ld t0, 32(sp)
        ld t1, 40(sp)
        ld t2, 48(sp)
        ld s0, 56(sp)
        ld s1, 64(sp)
        ld a0, 72(sp)
        ld a1, 80(sp)
        ld a2, 88(sp)
        ld a3, 96(sp)
        ld a4, 104(sp)
        ld a5, 112(sp)
        ld a6, 120(sp)
        ld a7, 128(sp)
        ld s2, 136(sp)
        ld s3, 144(sp)
        ld s4, 152(sp)
        ld s5, 160(sp)
        ld s6, 168(sp)
        ld s7, 176(sp)
        ld s8, 184(sp)
        ld s9, 192(sp)
        ld s10, 200(sp)
        ld s11, 208(sp)
        ld t3, 216(sp)
        ld t4, 224(sp)
        ld t5, 232(sp)
        ld t6, 240(sp)

        addi sp, sp, 256

        // return to whatever we were doing in the kernel.
        sret
```

#### Kernel trap handler

```c
// interrupts and exceptions from kernel code go here via kernelvec,
// on whatever the current kernel stack is.
// must be 4-byte aligned to fit in stvec.
void 
kerneltrap()
{
  int which_dev = 0;
  uint64 sepc = r_sepc();
  uint64 sstatus = r_sstatus();
  uint64 scause = r_scause();

  if((sstatus & SSTATUS_SPP) == 0)
    panic("kerneltrap: not from supervisor mode");
  if(intr_get() != 0)
    panic("kerneltrap: interrupts enabled");

// check if it’s an external interrupt or software interrupt,
// and handle it.
// returns 2 if timer interrupt,
// 1 if other device,
// 0 if not recognized.
  if((which_dev = devintr()) == 0){
    printf("scause %p\n", scause);
    printf("sepc=%p stval=%p\n", r_sepc(), r_stval());
    panic("kerneltrap");
  }

  // give up the CPU if this is a timer interrupt.
  if(which_dev == 2 && myproc() != 0 && myproc()->state == RUNNING)
    yield();

  // the yield() may have caused some traps to occur,
  // so restore trap registers for use by kernelvec.S's sepc instruction.
  w_sepc(sepc);
  w_sstatus(sstatus);
}
```

Xv6 sets a CPU’s `stvec` to `kernelvec` when that CPU enters the kernel from user space.

```c
void usertrap(void)
{
  // send interrupts and exceptions to kerneltrap(),
  // since we’re now in the kernel.
  w_stvec((uint64)kernelvec);

  // an interrupt will change sstatus &c registers,
  // so don’t enable until done with those registers.
  intr_on();
```

### Disable Interrupt

There is a small windows, that `stvec` hasn’t been set. It is critical that no device interrupt happen during that time.

RISC-V always disables interrupts when it starts to take a trap, and xv6 doesn’t enable them again until after it sets `stvec`.

```c
// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    printfinit();
    printf(“\n”);
    printf(“xv6 kernel is booting\n”);
    printf(“\n”);
    kinit();         *// physical page allocator*
    kvminit();       *// create kernel page table*
    kvminithart();   *// turn on paging*
    procinit();      *// process table*
    trapinit();      *// trap vectors*
    trapinithart();  // install kernel trap vector
    plicinit();      *// set up interrupt controller*
```

Note: 在设置好kernel trap vector to stvec之后 我们再enable interrupt，这是为了确保stvec的正确性。否则我们的stvec register在中断发生的时候 存的内容不正确可能会导致严重问题。

Set up to take exceptions and traps while in the kernel.

```c
void
trapinithart(void)
{
  w_stvec((uint64)kernelvec);
}
```

在正常情况下 从用户空间转到内核空间的时候 需要把Kernel handler 的地址写入存储器stvec

### When and where do we set up `stvec` if trap is from user?

其实在`usertrap handler`的c代码里 一开始就存了kernel trap handler进`stvec` register

```text
void usertrap(void)
{
  // send interrupts and exceptions to kerneltrap(),
  // since we’re now in the kernel.
  w_stvec((uint64)kernelvec);

  // an interrupt will change sstatus &c registers,
  // so don’t enable until done with those registers.
  intr_on();
```

这样其实只要在kernel mode, stvec都是一直指向kernel trap handler. 保证的逻辑的正确性！

