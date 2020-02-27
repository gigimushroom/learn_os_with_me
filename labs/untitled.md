---
description: Implement features relevant to memory-mapping a file
---

# Lab 9 mmap



## Introduction 

The `mmap` and `munmap` system calls allow UNIX programs to exert detailed control over their address spaces. They can be used to share memory among processes, to map files into process address spaces, and as part of user-level page fault schemes such as the garbage-collection algorithms discussed in lecture. In this lab you’ll add `mmap` and `munmap` to xv6, focusing on memory-mapped files.

The manual page \(run man 2 mmap\) shows this declaration for mmap:

```c
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
```

## TASK

Implement features relevant to memory-mapping a file. You can assume that addr will always be zero, meaning that the kernel should decide the virtual address at which to map the file. mmap returns that address, or 0xffffffffffffffff if it fails. length is the number of bytes to map; it might not be the same as the file’s length. prot indicates whether the memory should be mapped readable, writeable, and/or executable; you can assume that prot is PROT\_READ or PROT\_WRITE or both. flags will be either MAP\_SHARED, meaning that modifications to the mapped memory should be written back to the file, or MAP\_PRIVATE, meaning that they should not. You don’t have to implement any other bits in flags. fd is the open file descriptor of the file to map. You can assume offset is zero \(it’s the starting point in the file at which to map

munmap\(addr, length\) should remove mmap mappings in the indicated address range. If the process has modified the memory and has it mapped MAP\_SHARED, the modifications should first be written to the file. An munmap call might cover only a portion of an mmap-ed region, but you can assume that it will either unmap at the start, or at the end, or the whole region \(but not punch a hole in the middle of a region\).

See [Lab: mmap](https://pdos.csail.mit.edu/6.828/2019/labs/mmap.html)

## Solution

### Add system calls

Add `mmap` and `munmap` system calls and associated flags in the system.

### Define VMA \(virtual memory area\) Per Process

Define VMA to keep track what mmap has mapped for each process.

```c
struct vm_area_struct {
  int valid;
  uint64 start_ad;
  uint64 end_ad;
  int len;
  int prot;
  int flags;
  struct file *file;
  int fd;
};
```

Each Process has a fixed-size array of VMAs

```text
// Per-process state*
struct proc {
  …
  struct vm_area_struct vma[100];
  uint64 cur_max; // default to MAXVA - 2 * PGSIZE*
};
```

### How to determine VMA start and end address

We design the **initial max** virtual address of VMA is `MAXVA - 2 * PGSIZE`, which is 2 pages below the max. The reason behind is first 2 pages are trampoline and trap frame. The VMA list is allocated from top to bottom. The current max VA is adjusted after we create a new VMA. So next allocation’s end address can be set to current max. \[image:BF5B81DF-53B0-4E85-AA07-19C5BDF646BB-41823-0001AC129E733C96/Screen Shot 2020-02-27 at 11.10.56 AM.png\]

### `mmap` code

```c
  uint64 cur_max = p->cur_max;  
  uint64 start_addr = PGROUNDDOWN(cur_max - size);

  struct vm_area_struct *vm = 0;
  for (int I=0; I<100; I++) {
    if (p->vma[I].valid == 0) {
      vm = &p->vma[I];
      break;
    }
  }
  if (vm) {
    vm->valid = 1;
    vm->start_ad = start_addr;
    vm->end_ad = cur_max;
    vm->len = size;
    vm->prot = prot;
    vm->flags = flags;
    vm->fd = fd;
    vm->file = p->ofile[fd];
    vm->file->ref++;
    // reset process current max available
    p->cur_max = start_addr;
  } else {
    return 0xffffffffffffffff;
  }
  return start_addr;
```

### `mmap` permission

Check that mmap doesn’t allow read/write mapping of a file opened read-only.

```c
struct proc *p = myproc();
  struct file *f = p->ofile[fd];
  if (flags & MAP_SHARED) {
    if (!(f->writable) && (prot & PROT_WRITE)) {
      printf(“File is read-only, but we mmap with write permission and flag. %d vs %p\n”,
            f->writable, prot);
      return 0xffffffffffffffff;
    }
  }
```

### Lazy page allocation

#### idea

Fill in the page table lazily, in response to page faults. That is, mmap should not allocate physical memory or read the file. Instead, do that in page fault handling code in \(or called by\) usertrap, as in the lazy page allocation lab. The reason to be lazy is to ensure that mmap of a large file is fast, and that mmap of a file larger than physical memory is possible.

#### approach

Add code to cause a page-fault in a mmap-ed region to allocate a page of physical memory, read 4096 bytes of the relevant file into that page, and map it into the user address space.

#### code

In trap handler:

```text
if (r_scause() == 13 || r_scause() == 15) {
    printf(“usertrap(): mmap page fault %p (%s) pid=%d\n”, r_scause(), scause_desc(r_scause()), p->pid);

    // find which VMA has it.
    uint64 fault_addr = r_stval();
    struct vm_area_struct *vm = 0;
    for (int I=0; I<100; I++) {
      if (p->vma[I].start_ad <= fault_addr && 
          fault_addr <= p->vma[I].end_ad ) {
        vm = &p->vma[I];
        break;
      }
    }
    if (!vm) {
      printf(“VM addr is not lived in VMA, error out.\n");
      p->killed = 1;
      goto bad;
    }

    // allocate a page of physical memory.
    char *pa = kalloc();
      if(pa == 0)
        panic(“kalloc”);
    memset(pa, 0, PGSIZE);

    // Map it into the user address space.*
    // Install the page and ensure user can access. Use PTE_U.*
    uint64 fault_addr_head = PGROUNDDOWN(fault_addr);
    if (mappages(p->pagetable, fault_addr_head, PGSIZE, (uint64)pa, vm->prot | PTE_U) != 0) {
      kfree(pa);
      p->killed = 1;
    }

    // read 4096 bytes of the relevant file into user space.
    // IMPORTANT: the read offset is the distance between page_fault_addr and VMA->start Since file is mapped from offset 0. 
    int distance = fault_addr_head - vm->start_ad;
    mmap_read(vm->file, fault_addr_head, distance, PGSIZE);
```

The code is doing the following: 

1. Find which VMA owns the VA. Error out if not found. 

2. Allocate a new physical page. 

3. Map it into user address space, by installing to user page table. 

4. Read one page of the file starting from offset into user space.

Note: The file reading offset is `page_fault_addr - VMA->start`. The reason behind is that file is mapped from offset 0 to VMA-&gt;start. If we want to access some middle part of the file, starting from virtual address X, we have to find the distance, which is \(X - VMA-&gt;start\).

### Implement `munmap`

#### Approach

* find the VMA for the address range and unmap the specified pages \(hint: use uvmunmap\). 
* If munmap removes all pages of a previous mmap, it should decrement the reference count of the corresponding struct file.
  * If an unmapped page has been modified and the file is mapped MAP\_SHARED, write the page back to the file.

#### Code

**Find VMA to free**

```text
uint64 start_base = PGROUNDDOWN(addr);
  uint64 end_base = PGROUNDDOWN(addr + size);

  struct proc *p = myproc();
  struct vm_area_struct *vm = 0;
  for (int I=0; I<100; I++) {
    if (p->vma[I].valid == 1 && 
        p->vma[I].start_ad <=  start_base &&
        end_base <= p->vma[I].end_ad) {
      vm = &p->vma[I];
      break;
    }
  }
  if (!vm) {
    printf(“Cannot found VMA. start base(%p), end base(%p)\n”,
          start_base, end_base);
    return -1;
  }
```

**Write back if Shared flag is set**

```text
  if (vm->flags & MAP_SHARED) {
    printf(“….We need to write back file\n”);
    struct file *f = vm->file;
    begin_op(f->ip->dev);
    ilock(f->ip);
    writei(f->ip, 1, vm->start_ad, 0, vm->len);
    iunlock(f->ip);
    end_op(f->ip->dev);
  }
```

**Remove mappings from a page table**

```text
  pte_t *pte;
  for(int I = start_base; I <= end_base; I+=PGSIZE){
    if((pte = walk(p->pagetable, I, 0)) == 0) {
      if(*pte & PTE_V) {
        uvmunmap(p->pagetable, I, PGSIZE, 0);
      }
    }
  }
```

**Fix VMA metadata**

```text
  if (vm->start_ad == start_base && end_base < vm->end_ad) {
    vm->start_ad = end_base;
    vm->len -= size;
  } else if (vm->start_ad < start_base && end_base == vm->end_ad){
    // last part is un-map
    vm->end_ad = start_base - 1;
    vm->len = size;
  } else if (vm->start_ad == start_base && vm->end_ad == end_base) {
    // exact size
    vm->file->ref—;
    vm->valid = 0;
    vm->len = 0;
  } else if (vm->start_ad < start_base && end_base < vm->end_ad) {
    // Speical case un-map in middle. Not supported
  }
```

### Modify exit to unmap the process’s mapped regions as if munmap had been called

```c
void free_all_vma(pagetable_t pagetable, uint64 start, uint64 end) {
  pte_t *pte;
  for(int I = start; I <= end; I+=PGSIZE) {
    if((pte = walk(pagetable, I, 0)) == 0) {
      if(*pte & PTE_V) {
        uvmunmap(pagetable, I, PGSIZE, 0);
      }
    }
  }
}
```

Free all VMA areas

```text
struct vm_area_struct *vm = 0;
  for (int I=0; I<100; I++) {
    if (p->vma[I].valid == 0) {
      continue;
    }
    vm = &p->vma[I];
    vm->valid = 0;
    free_all_vma(p->pagetable, vm->start_ad,vm->end_ad);
  }
```

### Modify fork to ensure that the child has the same mapped regions as the parent.

Helper

```text
void copy_vma(struct vm_area_struct *dst, struct vm_area_struct *src) {
  dst->valid = 1;
  dst->start_ad = src->start_ad;
  dst->end_ad = src->end_ad;
  dst->len = src->len;
  dst->prot = src->prot;
  dst->flags = src->flags;
  dst->fd = src->fd;
  dst->file = src->file;
  dst->file->ref++;
}
```

Copy VMA areas from Parent to Child

```text
  for (int I=0; I<100; I++) {
    if (p->vma[I].valid == 1) {
      struct vm_area_struct *src = 0;
      src = &p->vma[I];
      // find one available from child.
      struct vm_area_struct *dst = 0;
      for (int j=0; j<100; j++) {
        if (np->vma[j].valid == 0) {
          dst = &np->vma[j];
          break;
        }
      }
      if (dst) {
        copy_vma(dst, src);
        dst->file = np->ofile[dst->fd];
      }
    }
  }
```

## Optional challenges:

* If two processes have the same file mmap-ed \(as in fork\_test\), share their physical pages. You will need reference counts on physical pages.
* Your solution probably allocates a new physical page for each page read from the mmap-ed file, even though the data is also in kernel memory in the buffer cache. Modify your implementation to use that physical memory, instead of allocating a new page. This requires that file blocks be the same size as pages \(set BSIZE to 4096\). You will need to pin mmap-ed blocks into the buffer cache. You will need worry about reference counts.
* Remove redundancy between your implementation for lazy allocation and your implementation of mmap-ed files. \(Hint: create a VMA for the lazy allocation area.\)
* Modify exec to use a VMA for different sections of the binary so that you get on-demand-paged executables. This will make starting programs faster, because exec will not have to read any data from the file system.
* Implement page-out and page-in: have the kernel move some parts of processes to disk when physical memory is low. Then, page in the paged-out memory when the process references it.

