# Start xv6 and the first process

1. RISC-V powers on, it runs a boot loader which is stored in read-only memory.
2. Boot loader loads xv6 kernel into memory.
3. Load kernel at physical address at 0x80000000, instead of 0x0. Because 0x0:0x8000000 contains I/O devices.
4. CPU executes xv6 starting at \_entry, to set up a stack so xv6 can run C code.
5. Each CPU has own stack.
6. Move stack pointer register sp points to stack0 + 4096, since stacks grows down.
7. \_entry calls into C code start\(\).
8. start\(\) perform configurations in machine mode.
9. start\(\) set return address to main\(\) by writing main’s address into register mepc.
10. start\(\) disable paging.
11. start\(\) initialize clock interrupts.
12. Then start\(\) enter supervisor mode by calling mret.
13. Program counter changes to main\(\).
14. main\(\) does all initialization devices and subsystems, set up address space, create kernel page table, allocate a page for the process’s kernel stack, etc.
15. main\(\) creates first process by calling userinit\(\).
16. The function loads an assembly file 'initcode.S’ and execute instructions. It re-enters kernel mode by invoking the exec system call.
17. exec\(\) replaces the memory and registers with new program /init
18. /init program create file descriptor 0, 1, 2, and starts a console shell.
19. Child is shell, parent loops and handles orphaned zombies repeatly.
20. The system is up.

[https://pdos.csail.mit.edu/6.828/2019/xv6/book-riscv-rev0.pdf](https://pdos.csail.mit.edu/6.828/2019/xv6/book-riscv-rev0.pdf)

