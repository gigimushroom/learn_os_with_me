---
description: Your task is to implement copy-on-write fork in the xv6 kernel
---

# Lab 5 Copy-on-Write Fork for xv6

## The problem

The fork\(\) system call in xv6 copies all of the parent process’s user-space memory into the child. If the parent is large, copying can take a long time. In addition, the copies often waste memory; in many cases neither the parent nor the child modifies a page, so that in principle they could share the same physical memory. The inefficiency is particularly clear if the child calls exec\(\), since exec\(\) will throw away the copied pages, probably without using most of them. On the other hand, if both parent and child use a page, and one or both writes it, a copy is truly needed.

## Task

Your task is to implement copy-on-write fork in the xv6 kernel

## The solution

The goal of copy-on-write \(COW\) fork\(\) is to defer allocating and copying physical memory pages for the child until the copies are actually needed, if ever.

{% hint style="info" %}
COW fork\(\) creates just a pagetable for the child, with PTEs for user memory pointing to the parent’s physical pages. COW fork\(\) marks all the user PTEs in both parent and child as not writable. 
{% endhint %}

Modify `uvmcopy` to do so. `fork` calls this function to allocate child memory.

```c
// Given a parent process’s page table, copy
// its memory into a child’s page table.
// Copies both the page table and the
// physical memory.
// returns 0 on success, -1 on failure.
// frees any allocated pages on failure.
int
uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, I;
  uint flags;
  for(I = 0; I < sz; I += PGSIZE){
    if((pte = walk(old, I, 0)) == 0)
      panic(“uvmcopy: pte should exist”);
    if((*pte & PTE_V) == 0)
      panic(“uvmcopy: page not present”);
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    // Record the page is COW mapping.
    flags |= PTE_RSW;
    // clear PTE_W in the PTEs of both child and parent*
    flags &= (~PTE_W);

    // map the parent’s physical pages into the child
    if(mappages(new, I, PGSIZE, (uint64)pa, flags) != 0){
      //kfree(mem);
      goto err;
    }
    // Bump the reference count*
    add_ref((void*)pa);
    // Remove parent page table mapping.
    uvmunmap(old, I, PGSIZE, 0);
    // Re-add the mapping with write bit cleared flags.
    if (mappages(old, I, PGSIZE, pa, flags) != 0) {
      goto err;
    }
  }
```

{% hint style="info" %}
When either process tries to write one of these COW pages, the CPU will force a page fault. The kernel page-fault handler detects this case, allocates a page of physical memory for the faulting process, copies the original page into the new page, and modifies the relevant PTE in the faulting process to refer to the new page, this time with the PTE marked writeable. When the page fault handler returns, the user process will be able to write its copy of the page. 
{% endhint %}

{% code title="trap.c usertrap handler:" %}
```c
else if (r_scause() == 15) {
// Modify usertrap() to recognize page faults.
// When a page-fault occurs on a COW page, allocate a new page with kalloc(),
// copy the old page to the new page,
// and install the new page in the PTE with PTE_W set.

  // Get physial page address and correct flags.
  uint64 start_va = PGROUNDDOWN(r_stval());
  pte_t *pte;
  pte = walk(p->pagetable, start_va, 0);
  if (pte == 0) {
    printf(“page not found\n”);
    p->killed = 1;
    goto end;
  }
  if ((*pte & PTE_V) && (*pte & PTE_U) && (*pte & PTE_RSW)) {
    uint flags = PTE_FLAGS(*pte);
    // +Write, -COW
    flags |= PTE_W;
    flags &= (~PTE_RSW);

    char *mem = kalloc();
    char *pa = (char *)PTE2PA(*pte);
    memmove(mem, pa, PGSIZE);
    uvmunmap(p->pagetable, start_va, PGSIZE, 0);
    // decrement old pa ref count.
    dec_ref((void*)pa);
    if (mappages(p->pagetable, start_va, PGSIZE, (uint64)mem, flags) != 0) {
      p->killed = 1;
      printf(“sometthing is wrong in mappages in trap.\n”);
    }
  }
  ...
```
{% endcode %}

COW fork\(\) makes freeing of the physical pages that implement user memory a little trickier. A given physical page may be referred to by multiple processes’ page tables, and should be freed only when the last reference disappears.

```c
void add_ref(void *pa) {
  int index = get_ref_index(pa);
  if (index == -1) {
    return;
  }
  refc[index] = refc[index] + 1;
}

void dec_ref(void *pa) {
  int index = get_ref_index(pa);
  if (index == -1) {
    return;
  }
  int cur_count = refc[index];
  if (cur_count <= 0) {
    panic(“def a freed page!”);
  }
  refc[index] = cur_count - 1;
  if (refc[index] == 0) {
    // we need to free page
    kfree(pa);
  }
}
```

Replace all the places calling `kfree` with `dec_ref`. 

Modify `kalloc` to call `add_ref`.

Modify `copyout()` to use the same scheme as page faults when it encounters a COW page.

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
    if(va0 >= MAXVA) {
      return -1;
    }

    pte_t *pte;
    pte = walk(pagetable, va0, 0);
    if (pte == 0) {
      return -1;
    }

    if ((*pte & PTE_V) == 0) {
      return -1;
    }
    if ((*pte & PTE_U) == 0) {
      return -1;
    }

    if (*pte & PTE_RSW) {
      pa0 = PTE2PA(*pte);
      uint flags = PTE_FLAGS(*pte);
      // +Write, -COW
      flags |= PTE_W;
      flags &= (~PTE_RSW);

      char *mem = kalloc();
      memmove(mem, (void*)pa0, PGSIZE);
      uvmunmap(pagetable, va0, PGSIZE, 0);
      dec_ref((void*)pa0);
      if (mappages(pagetable, va0, PGSIZE, (uint64)mem, flags) != 0) {
        panic(“sometthing is wrong in mappages in trap.\n”);
      }
    }
```

Reference:

[https://pdos.csail.mit.edu/6.828/2019/labs/cow.html](https://pdos.csail.mit.edu/6.828/2019/labs/cow.html)

Result

![](../.gitbook/assets/screen-shot-2020-02-28-at-3.42.25-pm.png)

## 心得

unix给用户端提供了很多接口。提供了方便但是谁也无法保证用户程序不会滥用。复制fork导致多余内存的浪费必须要解决，否则一个不断复制的程序可以耗尽资源。所以大师们设计出copy on write的方案, trade了一点点性能换来了资源的节约。



