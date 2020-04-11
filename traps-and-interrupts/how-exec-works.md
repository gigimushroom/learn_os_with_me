---
description: Split the flow into 2 phases.
---

# How exec\(\) works

## Load program and Allocate pages

1. Check ELF header
2. Load program into memory
3. Allocate 2 pages, one for user stack. Another for guard page.

![](../.gitbook/assets/image%20%2816%29.png)

{% hint style="info" %}
Note: this graph has a bug. The following 3 items should not exist:

`argv, argc, 0xFFFFFF`
{% endhint %}

## Prepare the new image

Push argument strings to stack. Prepare rest of stack.

Push the array of `argv[]` pointers to stack.

Set `argv` in a1 register. Set process info, including `pagetable`, `sz` Set `epc = elf.entry`, initial program counter to main initial stack pointer. return `argc`, in RSIC-V, this sets `argc` in register a0. So arguments `main(argc, argv)` is set.

The jump preparation is done. Program will go to `main`, and get args set properly.

#### TODO

understand ELF.

## system/kernel/trap

