# Part 3: How Page Table is used

## How Page Table is used?

### Terms:

* _Physical memory allocation_: Use buddy allocation algorithm to alloc/free memory.
* _Kernel has own page table._
* Each _process also has its own page table._
* Hardware MMU translates VM to PM, and each CPU processor has a _satp register_ to save root page-table page.

### Questions: 

How do we make sure process page tables and kernel’s page table not pointing to the same physical memory address? Are they all use satp to find address?

### Solution:

![](../../.gitbook/assets/image%20%2818%29.png)

* Each process, and kernel all have its own Page Table.
* Switching between user process or to/from kernel needs to set satp register to the address of the root page-table page.
* By using MMU, translate the virtual memory to physical memory address. \(See page tables Part 1 for details\)
* Note: satp register stores one root page table only. 

### **Ask yourself** 

1. How MMU work 

2. Kernel page table, direct mapping 

3. How to turn on page? By storing address of root page-table page to satp register 

4. Create process kernel stack, used for saving context 

5. Process has own page table 

6. Buddy allocator is alloc/free physical address 

7. How buddy allocator works? 

8. All allocation/free physical memory in os is done by buddy allocator. 

9. How to print page table? 3 nested level. 

10. How to do lazy page allocation in user space?

