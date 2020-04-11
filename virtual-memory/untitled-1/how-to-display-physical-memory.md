# How to display physical memory

in OS kernel code, a physical address is uint64, but it is often being cast as a void_, or char_, then carry around. I think this makes a lot of interface like memset, memcpy, etc more convenient to use.

```c
void vmprint_helper(pagetable_t *pagetable*, char* prefix, int level) {
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

void vmprint(pagetable_t *pagetable*)
{
  printf(“page table %p\n”, pagetable);
  vmprint_helper(pagetable, “..”, 2);
}
```

```text
page table 0x0000000087f6e000
..0: pte 0x0000000021fda801 pa 0x0000000087f6a000
....0: pte 0x0000000021fda401 pa 0x0000000087f69000
......0: pte 0x0000000021fdac1f pa 0x0000000087f6b000
......1: pte 0x0000000021fda00f pa 0x0000000087f68000
......2: pte 0x0000000021fd9c1f pa 0x0000000087f67000
..255: pte 0x0000000021fdb401 pa 0x0000000087f6d000
....511: pte 0x0000000021fdb001 pa 0x0000000087f6c000
......510: pte 0x0000000021fdd807 pa 0x0000000087f76000
......511: pte 0x000000002000200b pa 0x0000000080008000
init: starting sh
page table 0x0000000087f63000
..0: pte 0x0000000021fd7c01 pa 0x0000000087f5f000
....0: pte 0x0000000021fd7801 pa 0x0000000087f5e000
......0: pte 0x0000000021fd801f pa 0x0000000087f60000
......1: pte 0x0000000021fd741f pa 0x0000000087f5d000
......2: pte 0x0000000021fd700f pa 0x0000000087f5c000
......3: pte 0x0000000021fd6c1f pa 0x0000000087f5b000
..255: pte 0x0000000021fd8801 pa 0x0000000087f62000
....511: pte 0x0000000021fd8401 pa 0x0000000087f61000
......510: pte 0x0000000021fdbc07 pa 0x0000000087f6f000
......511: pte 0x000000002000200b pa 0x0000000080008000
```

Another reason cast uint64 to void\* is easy to print. \(see above code about how to print page tables\)

