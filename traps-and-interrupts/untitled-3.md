# What is trampoline

### What is it?

Trampoline page stores code to switch between user and kernel space. The code is mapped at the same virtual address \(TRAMPOLINE\) in user and kernel space so that it continues to work when it switches page tables.

Note: the pointers, page table, sp, etc are stored in `trapframe`. 

{% page-ref page="untitled-2.md" %}

In trampoline, we just store plain assembly source code.

### Where does trampoline page store?

RISC-V hardware doesn’t switch page tables during a trap, we need the user page table to include a mapping for the trap vector instructions that **stvec** points to. Further, the trap vector must switch satp to point to the kernel page table. In order to avoid a crash, the vector instructions \(stored in trampoline page\) must be **mapped at the same address in the kernel page table as in the user page table.**

![](../.gitbook/assets/image%20%2824%29.png)

![](../.gitbook/assets/image%20%2828%29.png)

In this way, after switching page table root in `satp` register, virtual memory is still the same, so it can continue to execute.

{% page-ref page="trap-home-page/" %}



