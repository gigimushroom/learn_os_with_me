---
description: Understand and improve physical memory allocator
---

# Lab 3 Buddy Allocator

## The Problem

xv6 has only a page allocator and cannot dynamically allocate objects smaller than a page. To work around this limitation, xv6 declares objects smaller than a page statically. For example, xv6 declares an array of file structs, an array of proc structures, and so on. As a result, the number of files the system can have open is limited by the size of the statically declared file array, which has NFILE entries \(see kernel/file.c and kernel/param.h\).

## Your Task

For this lab we have replaced the page allocator in the xv6 kernel with a buddy allocator. You will modify xv6 to use this allocator to allocate and free file structs so that xv6 can have more open file descriptors than the existing system-wide limit NFILE. Furthermore, you will implement an optimization that reduces the buddy’s use of memory.

### Task 1:

Modify kernel/file.c to use the buddy allocator so that the number of file structures is limited by memory rather than NFILE.

### Task 2:

The buddy allocator is space inefficient. The alloc array has a bit for each block for each size. There is a clever optimization that reduces the cost to only one bit for each pair of blocks. This single bit is B1\_is\_free XOR B2\_is\_free, for a buddy pair of blocks B1 and B2. Each time a block is allocated or freed, you flip the bit to reflect the change. For example, if B1 and B2 are allocated, the bit will be zero and if B1 is freed the bit changes to 1. If the bit is 1 and B2 is freed, then we know that B1 and B2 should be merged. Saving 1/2 bit per block matters when xv6 uses the buddy allocator for the roughly 128 Mbyte of free memory that xv6 must manage: this optimization saves about 1 MByte of memory.

## Solution

### Allocate file use buddy allocator

```c
struct file*
filealloc(void)
{
  struct file *f;

  acquire(&ftable.lock);
  char *p = bd_malloc(sizeof(ftable.file));
  f = (struct file*) p;
  f->ref = 1;
  release(&ftable.lock);
  return f;
}
```

### Close file should release memory

```text
void
fileclose(struct file *f)
{
…
bd_free(f);
```

### Only one bit for each pair of blocks

#### Add flip bit helpers

```c
void flip_bit(char *array, int index) {
  index >>= 1; // index /= 2
  // find bit position from 8 total
  char m = (1 << (index % 8)); 
  // find which char we belongs to, then xor with 1, so flip.
  array[index/8] ^= m;
}

int bit_alloc_get(char *array, int index) {
  index >>= 1;
  char b = array[index/8];
  char m = (1 << (index % 8));
  return (b & m) == m;
}
```

#### Replace `bit_set` usage with `flip_bit`

#### Replace `bit_clear` usage with `flip_bit`

#### Replace `bit_isset` usage with `bit_alloc_get`

#### Add in range check

`#define in_range(a, b, x) (((x) >= (a)) && ((x) < (b)))`

#### Change how buddy allocation initialize

**Code Walk Through**

During initialization, buddy allocator has a range **\[free\_start, free\_end\)** `bd_init` calls`int free = bd_initfree(p, bd_end);` to initialize free lists for each size k. `bd_initfree` calls `bd_initfree_pair`, which initialize the free lists for each size k. For each size k, there are only two pairs that may have a buddy that should be on free list: bd\_left and bd\_right.

**Fix bd\_initfree\_pair**

Calling this function, passed in the free range.

```c
free += bd_initfree_pair(k, left, bd_left, bd_right);
free += bd_initfree_pair(k, right, bd_left, bd_right);
```

`bd_initfree_pair` checks if set and in range uses new helpers. If buddy is in free range, add to free list, otherwise, add current index to free list.

```c
// If a block is marked as allocated and the buddy is free, put the
// buddy on the free list at size k.
int
bd_initfree_pair(int k, int bi, void *free_start, void *free_end) {
  int buddy = (bi % 2 == 0) ? bi+1 : bi-1;
  int free = 0;
  if(bit_alloc_get(bd_sizes[k].alloc, bi)) {
    // one of the pair is free*
    free = BLK_SIZE(k);
    if(in_range(free_start, free_end, addr(k, buddy))) {
      lst_push(&bd_sizes[k].free, addr(k, buddy));   // put buddy on free list
    }
    else
      lst_push(&bd_sizes[k].free, addr(k, bi));      // put bi on free list
  }
  return free;
}
```

#### Reduce the bit size

In `bd_init`, change from `sz = sizeof(char)* ROUNDUP(NBLK(k), 8)/8;` To: `sz = sizeof(char)* ROUNDUP(NBLK(k), 16)/16; // Twins use 1 single bit!` Hence, the total size of each block is reduced in half.

## Next Step?

## 心得

伙伴算法的初始化有多种方式, xv6的实现非常精妙。我花费了很多时间理解大师的思想，最后发现**可视化**是最佳理解方法了。花了很多图，最终一目了然。

{% page-ref page="../virtual-memory/untitled-1/" %}



