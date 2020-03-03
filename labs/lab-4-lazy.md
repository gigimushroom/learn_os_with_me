---
description: Allocate user memory lazily
---

# Lab 4 Lazy

## Introduction 

One of the many neat tricks an O/S can play with page table hardware is lazy allocation of user-space heap memory. Xv6 applications ask the kernel for heap memory using the sbrk\(\) system call. In the kernel we’ve given you, sbrk\(\) allocates physical memory and maps it into the process’s virtual address space. However, there are programs that use sbrk\(\) to ask for large amounts of memory but never use most of it, for example to implement large sparse arrays. 

To optimize for this case, sophisticated kernels allocate user memory lazily. That is, sbrk\(\) doesn’t allocate physical memory, but just remembers which addresses are allocated. When the process first tries to use any given page of memory, the CPU generates a page fault, which the kernel handles by allocating physical memory, zeroing it, and mapping it.

## Your Task

Add this lazy allocation feature to xv6

## Task 1: Print page table

### Requirements

Implement a function that prints the contents of a page table. Define the function in kernel/vm.c; it has the following prototype: `void vmprint(pagetable_t)`. Insert a call to vmprint in exec.c to print the page table for the first user process.

The output of vmprint for the first user-level process should be as follows:

```c
page table 0x0000000087f6e000
 ..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
 .. ..0: pte 0x0000000021fda401 pa 0x0000000087f69000
 .. .. ..0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
 .. .. ..1: pte 0x0000000021fda00f pa 0x0000000087f68000
 .. .. ..2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
 ..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
 .. ..511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
 .. .. ..510: pte 0x0000000021fdd807 pa 0x0000000087f76000
 .. .. ..511: pte 0x000000002000200b pa 0x0000000080008000
```

* The first line prints the address of the argument of `vmprint`. 
* Each PTE line shows the PTE index in its page directory, the pte, the physical address for the PTE. 
* The output should also indicate the level of the page directory: the top-level entries are preceded by “..”, the next level down with another “..”, and so on. 
* You should not print entries that are not mapped. 
* In the above example, the top-level page directory has mappings for entry 0 and 255. The next level down for entry 0 has only index 0 mapped, and the bottom-level for that index 0 has entries 0, 1, and 2 mapped.

### Solution

```c
char* print_prefix(int level) {
  if (level == 2) {
    return “..”;
  } 
  else if (level == 1) {
    return “ .. ..”;
  }
  return “ .. .. ..”;
}

void vmprint_helper(pagetable_t pagetable, char* prefix, int level) {
  // there are 2^9 = 512 PTEs in a page table.
  for(int k = 0; k < 512; k++) {
    pte_t pte = pagetable[k];
    if(pte & PTE_V) {
      printf(“%s%d: pte %p pa %p\n”, prefix, k, pte, PTE2PA(pte));
      int next_level = level - 1;
      if (level > 0) {
        uint64 child = PTE2PA(pte);
        pagetable_t pt = (pagetable_t)child;
        vmprint_helper(pt, print_prefix(next_level), next_level);
      }
    }
  }
}

void vmprint(pagetable_t pagetable)
{
  printf(“page table %p\n”, pagetable);
  vmprint_helper(pagetable, “ ..”, 2);
}
```

## Task 2: Eliminate allocation from sbrk\(\)

### Requirement

Your new sbrk\(n\) should just increment the process’s size \(myproc\(\)-&gt;sz\) by n and return the old size. It should not allocate memory

### Solution

```c
uint64
sys_sbrk(void)
{
  int addr;
  int n;

  if(argint(0, &n) < 0) {
    return -1;
  }
  addr = myproc()->sz;
  myproc()->sz+=n;
  if (n < 0) {
    uvmdealloc(myproc()->pagetable, addr, myproc()->sz);
  }
  return addr;
}
```

## Task 3: Lazy allocation

### Requirement

Modify the code in trap.c to respond to a page fault from user space by mapping a newly-allocated page of physical memory at the faulting address, and then returning back to user space to let the process continue executing.

{% hint style="info" %}
Check whether a fault is a page fault by seeing if `r_scause()` is 13 or 15 in `usertrap()`.
{% endhint %}

{% hint style="info" %}
Look at the arguments to the printf\(\) in `usertrap()` that reports the page fault, in order to see how to find the virtual address that caused the page fault.
{% endhint %}

{% hint style="info" %}
Use `PGROUNDDOWN(va)` to round the faulting virtual address down to a page boundary.
{% endhint %}

### Solution

In `trap.c: usertrap`

```c
else if (r_scause() == 13 || r_scause() == 15) {
    // 13: Page load fault, 15: Page store fault
    if (r_stval() >= p->sz) {
      p->killed = 1;
      goto end;
    }

    if (r_stval() < p->ustack) {
      printf(“Access guard page is invalid.\n”);
      p->killed = 1;
      goto end;
    }

    // round vm page to page boundary
    uint64 vm = PGROUNDDOWN(r_stval());

    char *pa = kalloc();
    if(pa == 0) {
      p->killed = 1;
      goto end;
    }
    memset(pa, 0, PGSIZE);
    // install page to page table
    if (mappages(p->pagetable, vm, PGSIZE, (uint64)pa, PTE_W|PTE_R|PTE_X|PTE_U) != 0) {
      kfree(pa);
      p->killed = 1;
    }
  }
```

Now trap handler will allocate a new physical memory and install to user page table once passed safety checks.

`p->ustack` is new added to Process struct, indicate the bottom of user stack. User stack is established in `exec.c` when loading file image to execute. See below code for references:

```c
// Allocate two pages at the next page boundary.*
// Use the second as the user stack.*
  sz = PGROUNDUP(sz);
  if((sz = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;
  p->ustack = stackbase;  // ADD THIS ONLY
```

#### Reference: User Process Address Space

![](../.gitbook/assets/screen-shot-2020-02-28-at-11.24.18-am.png)

_If accessing guard page, we should fail!_

#### Fix a couple of place in `vm.c` so no panic

In `uvmunmap`, if a PTE does not exist, continue the loop instead of panic. If an PA in PTE not mapped, continue the loop instead of panic. Same fix in `uvmcopy`

#### Fix Copy in/out between kernel and user space

**Requirement**

Handle the case in which a process passes a valid address from sbrk\(\) to a system call such as read or write, but the memory for that address has not yet been allocated.

**Solution**

`copyout` : Copy from kernel to user. `copying`: Copy from user to kernel. They originally will error out if physical address \(PA\) does not exist. We should fix it, so allow to continue looping if such PA does not exist. Since the virtual address could be valid, so trap handler will allocate new pages later. We should allow the copy to continue.

Change from:

```c
while (len > 0) {
…
if(pa0 == 0)
  return -1;
…
}
```

Change to:

```c
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;

    if(pa0 != 0)
      memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

```c
// Copy from user to kernel.
// Copy len bytes to dst from virtual address srcva in a given page table.
// Return 0 on success, -1 on error.
int
copyin(pagetable_t pagetable, char *dst, uint64 srcva, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    n = PGSIZE - (srcva - va0);
    if(n > len)
      n = len;
    if (pa0 != 0) {
      // In order to handle the case in which a process passes a valid address
      // from sbrk() to a system call such as read or write,
      // but the memory for that address has not yet been allocated.
      // Hence, only memmove() if pa0 is not NULL.
      memmove(dst, (void *)(pa0 + (srcva - va0)), n);
    }

    len -= n;
    dst += n;
    srcva = va0 + PGSIZE;
  }
  return 0;
}
```

**Result**

## 心得

中断是操作系统的核心，利用中断来做Lazy allocation也是精妙的思想。最终要解决的问题是不要过多分配完全不使用的内存给用户。用多少拿多少。

