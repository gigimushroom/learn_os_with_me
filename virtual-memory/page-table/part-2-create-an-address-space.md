# Part 2: Create an Address Space

## Create an Address Space

### Create a direct-map page table for kernel

There is a walk\(\) to find PTE for a given virtual address mappages\(\) installs PTE for new mappings

In kvminit\(\), first allocate a page of physical memory to hold the root page-table page. Then calls to install a bunch of things that kernel needs, including: instructions, data, physical memory range, and memory range for devices.

The install mapping `mappages` works as follow: 1. For given virtual address, find the pte 2. Convert physical address to pte and set permission bits 3. Return success or not.

`static pte_t * walk(pagetable_t pagetable, uint64 va, int alloc)`

1. Returns the address of the PTE in page table pagetable that corresponds to a virtual address.
2. Loop from level 2 to level 0
3. Find the pte entry using level, and va from current page table pageable.
4. Find page that pte points to.
5. If pte not exist or page not valid, allocate a new page.
6. End of loop.
7. Return the pte entry.

### Turn on Paging

Write physical address of the root page-table page into per CPU register satp. After this, CPU will begin translate addr using kernel page table. Flush the TLB to clear cache.

### Create Process Table

1. For process p, allocate 1 pages using buddy allocator for this process’s kernel stack.
2. Map it high in memory with permission X \| W.
3. Set it to be followed by an invalid guard page. This virtual page does not allocated in physical memory, it just occupies some space in process VM address space. Any access to those will trigger kernel panic.

   \`\`\`

   /// map kernel stacks beneath the trampoline,/

   /// each surrounded by invalid guard pages./

   **define KSTACK\(p\) \(TRAMPOLINE - \(\(p\)+1\)** _**2**_**PGSIZE\)**

// Allocate a page for the process's kernel stack./ // Map it high in memory, followed by an invalid/ // guard page char \*pa = kalloc\(\); if\(pa == 0\) panic\(“kalloc”\); uint64 va = KSTACK\(\(int\) \(p - proc\)\); kvmmap\(va, \(uint64\)pa, PGSIZE, PTE\_R \| PTE\_W\); p-&gt;kstack = va;

\`\`\`

1. Reloads the kernel page table into satp so that hardware knows about the new PTEs. \(Including flush TLB buffer cache\)

### Process has its own Page Table.

Different than the kernel one. The root page-table page needs to set in satp register when CPU is currently running this process. Process struct saves kernel root page-table address, so allows to switch to/from kernel.

\*\*\[\[Page Tables \(part 3\) - How Page Table is used?\]\]

## system/kernel/memory

