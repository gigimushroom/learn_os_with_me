# 系统调用的核心原理

### 3 kinds of traps:

1. system call
2. something illegal
3. device interrupts

   具体的执行方式详见\[\[how system calls get into / out of the kernel\]\].

trap from user space 比trap from kernel space难度大。 首先RISC-V Hardware不会switch page table, 我们要自己实现。

### Stack pointer也不正确, 不能依赖。

## 核心原理

**为了解决这些问题 xv6用了一个user page to map the trap vector instructions. 将这个特殊page的address 存给了stvec register.**

**这个传奇的user page就是大名鼎鼎的跳板page trampoline page.**

**Xv6 maps the trampoline page at the same virtual address in the kernel page table and in every user page table**

这样就算切换到了kernel page table, 我们一样可以继续执行trap vector instruction

process还用了trapframe存了重要的信息,以及足够空间store registers. 

It contains pointers to the current process’s kernel stack, the current CPU’s hartid, the address of usertrap, and the address of the kernel page table. uservec retrieves these values, switches satp to the kernel page table, and calls usertrap.

### 重要细节

`stvec` — ecall jumps here in kernel; address of trampoline 

`stvec`: The kernel writes the address of its trap handler here

The RISC-V jumps here to handle a trap. Risc-V provides a separate opcode to call the operating system. This is the `ECALL` instruction.

`static uint64 (*syscalls[28])(void) syscalls` is an 28 elements of array of pointers point to function.

```c
static uint64 (**syscalls*[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
…
```

[c - Need help to understand the syntax in xv6 kernel - Stack Overflow](https://stackoverflow.com/questions/28728848/need-help-to-understand-the-syntax-in-xv6-kernel) [Using the GNU Compiler Collection \(GCC\): Designated Inits](https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html) 

总结： 任何一个syscall在User space都是汇编。详见 usys.pl

```text
sub entry {
    my $name = shift;
    print “.global $name\n”;
    print “${name}:\n”;
    print “ li a7, SYS_${name}\n”;
    print “ ecall\n”;
    print “ ret\n”;
}

entry(“fork”);
entry(“exit”);
entry(“wait”);
entry(“pipe”);
…
```

普通user program例如echo, cat等会call `write` syscall. 

write其实一段生成的汇编代码\(generate code如上\)。将名字作为参数存入a7 register。 

call RISC-V ecall instruction。 

这个ecall会直接读取stvec的内容并且跳转。并且进入OS kernel模式。 

uservec在OS初始化时会设置好，作为trap的入口。 

所以进入uservec，做好一系列准备工作 进入C traphandler。 

如果trap是来自system call 就会根据a7的数字 呼叫相应system call.

`uint64 (*syscalls[])(void)` syscalls is array of pointers point to function. 

`{ [SYS_fork] sys_fork,...}` 是古老的语法 其实就是 \[a\]=b,表示当index是a的时候return b.

#### Open questions

a0 register到底存了什么 为什么和sscratch要swap？ 而且为什么要换呢 为什么不能直接用sscratch offset来存registers呢 example: instead of

```text
save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
…
```

Can we do

```text
save the user registers in TRAPFRAME
        sd ra, 40(sscratch)
        sd sp, 48(sscratch)
        sd gp, 56(sscratch)
        sd tp, 64(sscratch)
        sd t0, 72(sscratch)
```

