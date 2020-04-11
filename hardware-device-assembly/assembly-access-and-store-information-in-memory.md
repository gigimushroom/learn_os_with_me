# Assembly: Access and Store information in Memory

### Sizes of C data types in X86-64

![](../.gitbook/assets/image%20%2818%29.png)

### Operand forms in X86-64

![](../.gitbook/assets/image%20%2812%29.png)

`M[Addr]` means a reference to the value stored in memory starting at address `Addr`.

The memory of address `Addr` could come from user space, CPU and MLU works together to get the physical page \(PA\) from Page Table, then access the PA in RAM.

### RISC-V Integer Instructions

![](../.gitbook/assets/image%20%2837%29.png)

The operand forms is basically the same. `lb D, S` `sb S, D`

Example: `sd ra,40(sp)` Save 8 bytes \(double\) from `ra` register to memory address of \(sp + 40\).

