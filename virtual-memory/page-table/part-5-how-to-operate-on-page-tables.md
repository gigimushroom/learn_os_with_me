# Part 5: How to operate on page tables

This _section_ explains 3 things: 

1. How page table directory is stored. 

2. How to find the address of PTE given virtual address 

3. How to map page by creating PTE.

### How page table directory is stored.

```text
typedef uint64 pte_t;
typedef uint64 *pagetable_t;// 512 PTEs
```

pagetable\_t is a pointer points to 512 PTEs. page tables are physical memory, initialized by kalloc\(\). The \*pte stores physical memory.

### How to find the address of PTE given virtual address

```text
*// Return the address of the PTE in page table pagetable*
*// that corresponds to virtual address va.  If alloc!=0,*
*// create any required page-table pages.*
*//*
*// The risc-v Sv39 scheme has three levels of page-table*
*// pages. A page-table page contains 512 64-bit PTEs.*
*// A 64-bit virtual address is split into five fields:*
*//   39..63 — must be zero.*
*//   30..38 — 9 bits of level-2 index.*
*//   21..39 — 9 bits of level-1 index.*
*//   12..20 — 9 bits of level-0 index.*
*//    0..12 — 12 bits of byte offset within the page.*
static pte_t *
walk(pagetable_t *pagetable*, uint64 *va*, int *alloc*)
{
  if(va >= MAXVA)
    panic(“walk”);

  for(int level = 2; level > 0; level—) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

_`pagetable = (pde_t)kalloc())`_ 

Above code creates the level\_X page table directly from physical memory, and set to pte.

### How to map page.

```c
*// Create PTEs for virtual addresses starting at va that refer to*
*// physical addresses starting at pa. va and size might not*
*// be page-aligned. Returns 0 on success, -1 if walk() couldn’t*
*// allocate a needed page-table page.*
int
mappages(pagetable_t *pagetable*, uint64 *va*, uint64 *size*, uint64 *pa*, int *perm*)
{
  uint64 a, last;
  pte_t *pte;

  a = PGROUNDDOWN(va);
  last = PGROUNDDOWN(va + size - 1);
  for(;;){
    if((pte = walk(pagetable, a, 1)) == 0)
      return -1;
    if(*pte & PTE_V)
      panic("remap");
    *pte = PA2PTE(pa) | perm | PTE_V;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```

