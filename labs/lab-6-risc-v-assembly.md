---
description: warmup and learn assembly
---

# Lab 6 RISC-V assembly

## Warmup: RISC-V assembly 

There is a file user/call.c in your xv6 repo. **make fs.img** builds a user program call and a readable assembly version of the program in user/call.asm. Read the code in call.asm for the functions g, f, and main. The instruction manual for RISC-V is in the doc directory \(doc/riscv-spec-v2.2.pdf\).

### Questions and Answers

**Which registers contain arguments to functions? For example, which register holds 13 in main’s call to printf?** 

a0–a7 and fa0-fa7contains function arguments. a0 is also return values. 13 is stored in register a2

**Where is the function call to f from main? Where is the call to g? \(Hint: the compiler may inline functions.\)** 

Call to f from main `26: 45b1 li a1,12` Call to g from f `14: 250d addiw a0,a0,3`

**At what address is the function printf located?** 

`0000000000000650 <printf>` 

![](../.gitbook/assets/image%20%2825%29.png)

Or see the following code in main:

```text
  30:    00000097              auipc    ra,0x0
  34:    620080e7              jalr    1568(ra) # 650 <printf>
```

program counter\(pc\) is `0x30`. 

`auipc` is to add 0x0 to pc, and store the result in ra. so ra is `0x30` 

`jalr` is jump and link register so 1568\(ra\) is: 

`hex(1568) + 0x30 => 0x620 + 0x30 = 0x650`

**What value is in the register ra just after the jalr to printf in main?**

![](../.gitbook/assets/image%20%2820%29.png)

`pc+4` is written to register ra. In our example asm, ra stores 0x38

## Reference

### Assembly Details

```text
int g(int x) {
   0:    1141                    addi    sp,sp,-16
   2:    e422                    sd    s0,8(sp)
   4:    0800                    addi    s0,sp,16
  return x+3;
}
   6:    250d                    addiw    a0,a0,3
   8:    6422                    ld    s0,8(sp)
   a:    0141                    addi    sp,sp,16
   c:    8082                    ret

void main(void) {
  1c:    1141                    addi    sp,sp,-16
  1e:    e406                    sd    ra,8(sp)
  20:    e022                    sd    s0,0(sp)
  22:    0800                    addi    s0,sp,16
  printf("%d %d\n", f(8)+1, 13);
  24:    4635                    li    a2,13
  26:    45b1                    li    a1,12
  28:    00000517              auipc    a0,0x0
  2c:    7d050513              addi    a0,a0,2000 # 7f8 <malloc+0xea>
  30:    00000097              auipc    ra,0x0
  34:    620080e7              jalr    1568(ra) # 650 <printf>
  exit(0);
  38:    4501                    li    a0,0
  3a:    00000097              auipc    ra,0x0
  3e:    27e080e7              jalr    638(ra) # 2b8 <exit>
```

#### ADDI

Add immediate \(with overflow\) 

Description: Adds a register and a sign-extended immediate value and stores the result in a register Operation: `$t = $s + imm; advance_pc (4);` 

Syntax: `addi $t, $s, imm`

#### sd

store a double world. Store 64 bits from s0 register to sp+8

![](../.gitbook/assets/image%20%2833%29.png)

## 心得

汇编在现代社会不再是必需品。太多高级语言包装了一切，蒙蔽了最初的最原始的思想，可是无论装饰再如何精彩，都遮挡不了最初最美的风景。那里才是我的追求。

