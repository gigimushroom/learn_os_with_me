# xv6 buddy allocator

### 本质：

1. **1. 分配出大小最为合适的连续物理内存**

   **2. 释放会合并周围的伙伴变成更大连续内存**

   **3. 分配更节约cost，减少fragmentation**

   **4. 释放速度更快**

### _Buddy allocation initialization_

```text
=== buddy allocator state ===
size 0 (16): free list:
  alloc: 0 0 0 0
size 1 (32): free list:
  alloc: 0 0
  split: 0 0
size 2 (64): free list:  0x109756000
  alloc: 0
  split: 0
```

NSIZES is number of entries in sizes array. Here is 3. MAXSIZE is NSIZES - 1. Largest index in sizes array. Here is 2. Number of Block at size k: 2 ^ \(MAXSIZE - k\). Here: k=0, NBLK=2^2=4 k=1, NBLK=2^\(2-1\)-2. k=2, NBLK= 2^0=1 Note: In this way, the last size item is guaranteed to have 1 block.

```c
struct sz_info {
  struct bd_list free;
  char *alloc;
  char *split;
};
typedef struct sz_info Sz_info;
static Sz_info bd_sizes[NSIZES];
```

bd\_sizes is the buddy allocator array. Each item is sz\_info. sz\_info contains a free list, an array alloc, and a split array. In bd\_init\(\): 1. Call mmap to allocate a heap size of memory. Save ptr to bd\_base. 2. For \[0, NSIZES\) A\) initialize every free list B\) allocate alloc array with correct size. \(Use `1 << (MAXSIZE-k)` to find number of block\) C\) memset to 0 since malloc does not clear bit. 3. For \[1, NSIZES\) A\) allocate split array with correct size. \(Use `1 << (MAXSIZE-k)` to find number of block\) C\) memset to 0 since malloc does not clear bit. 4. Push the bd\_base\(allocated from mmap\) to the `bd_sizes[MAXSIZE].free`. So the available free list is shown in biggest size group. 5. End of initialization.

In xv6 unix, the imp is slightly different. We get memory range \(Start, end\) from kernel. We mark the meta as allocated, mark the unreachable one as allocated \(HEAPSIZE - end\). Then for remaining free memory \(P to end\), we try to divide them into each size k free list, instead of putting all into the MAXSIZE free list.

### _Allocation_

What happen when calling `bd_malloc(8)`

```c
=== buddy allocator state ===
size 0 (16): free list:  0x109756010
  alloc: 1 0 0 0
size 1 (32): free list:  0x109756020
  alloc: 1 0
  split: 1 0
size 2 (64): free list:
  alloc: 1
  split: 1
```

1. Find a block that is &gt;= needed size \(here is 8 bytes\). This block is named as fk.
2. Find a free block\(k\) that is &gt;= the smallest block\(fd\). Return NULL if none.
3. Once found, pop the item\(p\) from its list \(bd\_sizes\[k\].free\).
4. Find block index of the item\(p\), set its alloc array corresponding bit.
5. Use a while loop to do the potentially split.
6. `for (; k > fk, k—)`

   A\) Find 2nd half of p. Named it q.

   B\) Find p’s index in k row. Mark k’s split array bit to set.

   C\) Find p’s index in k-1 row. Set its index in k-1’s alloc array bit.

   D\) Push the 2nd half\(q\) to k-1 row’s free list.

7. Return p, which is the ptr points to allocated item. It should be in the fk row.

> The key idea is once we found a matching block item, we split it to half. The first half is allocated \(think of moving this half to lower size\), and will be return to caller. The 2nd half is added to lower level's free list. Use a loop to keep split until reaching end\(smallest needed size\). In above example, after splits, size 2 has no free list. Size 1 has old size 2's 2nd half.
>
> ### Size 0 free list has ‘old size 2’s 1st half’s 2nd half’. The very 1st piece is allocated and returns to caller!
>
> There are a few of helpers worth learning:
>
> ```c
> // find first k such that 2^k >= n
> int firstk(size_t n) {
>   int k = 0;
>   size_t size = LEAF_SIZE;
>   while (size < n) {
>     k++;
>     size *= 2;
>   }
>   return k;
> }
> ```
>
> \`\`\`c // Compute the block index for address p at size k int blk\_index\(int k, char _p\) { int n = p - \(char_ \) bd\_base; return n / BLK\_SIZE\(k\); }

// Convert a block index at size k back into an address void _addr\(int k, int bi\) { int n = bi_  BLK\_SIZE\(k\); return \(char \*\) bd\_base + n; }

```text
*The entire memory is reflected in each block size array.*
```c
Size 0: |16|16|16|16|16|16|16|16|
Size 1: |32   |32   |32   |32   |
Size 2: |64                     |
```

But the actual ownership of each small piece of content only belongs to one size array, indicated by alloc bit, split bit. In this way, given size k, and address, we can find block index. Given block index, and size k, we can find address.

### The free list also operates on this big chunk of memory, but it is using double linked list.

### Memory Free

`bd_free(void *p)` 1. Find the size of block that p points to. If the block index of p is set in \(k + 1\)'s split bit array, it means the parent was split because of me, thus, k is the matching block. 2. Loop matching block, while k &lt; MAXSIZE 1. Find block index of p in current block k. 2. Clear alloc bit. We got a free block! 3. Find my buddy. 4. If my buddy is still allocated, loop breaks. 5. If not, continue. 6. Find address of my buddy. 7. Remove buddy from current free list. We are going to merge! 8. Find who is leader between me and buddy. Reset p if buddy is leader. 9. Find block index of p in k + 1. Clear our parent \(k + 1\)’s split bit array. 10. Current iteration ends.

1. Push the pointer p to current k’s free list.

Note:

* The loop will only end either p’s buddy is still allocated or reach MAXSIZE. So the above step 3 is either pushing the single freed p to free list, or push the merged \(p+buddy\) to free list.
* Largest size block array’s alloc bit is always set.

### Optimization

Proof of concept: Find index, find address for malloc have to cut to half.

```c
// Compute the block index for address p at size k/
int
blk_index_malloc(int k, char *p) {
  int n = p - (char *) bd_base;
  return n / BLK_SIZE(k) / 2;
}

// Convert a block index at size k back into an address/
void *addr_malloc(int k, int bi) {
  int n = bi * BLK_SIZE(k) / 2;
  return (char *) bd_base + n;
}
```

In bd\_malloc, Find bit index of p, flips the bit. Case 1: If current bit is 0, means both are free. By changing result to 1. So we have 1 free, 1 allocated. Case 2: If current xor bit is 1, means the other buddy is allocated, we are allocated the 2nd, so both of them will be allocated, we got 1 xor 1 which is 0.

In bd\_free, If current bit is 0, it means both buddy and p are allocated. Loop needs to break, but flip the bit before the break. If current bit is 1, it means we are the allocated one, my buddy is free, flipt the bit\(set to 0\), indicating all free, continue the merge process.

```c
// allocating memory for alloc array for each block size
int sz = sizeof(char)*ROUNDUP(NBLK(k), 8)/8;
bd_sizes[k].alloc = malloc(sz);
```

We want to allocate half of the size, so we want to change above to: `int sz = sizeof(char)*ROUNDUP(NBLK(k) , 16)/16;`

Let’s give above a try!!!! It works as a proof of concept.

_There are some difference between simple buddy allocation implementation and the xv6 buddy impl._ In simple one, during initialization, the whole free memory is set in max\_size \(last row\) as free list. First allocation will split from the highest size, all the way to the desired size. _In xv6 one, it tries to initialize the free lists for each size k!_

```c
/// If a block is marked as allocated and the buddy is free, put the/
/// buddy on the free list at size k./
int
bd_initfree_pair(int k, int bi, void *free_start, void *free_end) {
  int buddy = (bi % 2 == 0) ? bi+1 : bi-1;
  int free = 0;
  if(bit_get(bd_sizes[k].alloc, bi)) {
    /// one of the pair is free/
    free = BLK_SIZE(k);
    if(in_range(free_start, free_end, addr(k, buddy)))
      lst_push(&bd_sizes[k].free, addr(k, buddy));   /// put buddy on free list/
    else
      lst_push(&bd_sizes[k].free, addr(k, bi));      /// put bi on free list/
  } else {
    printf("k(%d) is skipped for index(%d)!\n", k, bi);
  }
  return free;
}

// Initialize the free lists for each size k.  For each size k, there/
// are only two pairs that may have a buddy that should be on free list:/
// bd_left and bd_right./
int
bd_initfree(void *bd_left, void *bd_right) {
  int free = 0;

  for (int k = 0; k < MAXSIZE; k++) {   /// skip max size/
    int left = blk_index_next(k, bd_left);
    int right = blk_index(k, bd_right);
    int cache = bd_initfree_pair(k, left, bd_left, bd_right);
    free+=cache;
    printf(“k(%d), left index(%d), free size(%d)\n”, k, left, cache);
    if(right <= left) /// right == left means at highest level!/
      continue;
    cache = bd_initfree_pair(k, right, bd_left, bd_right);
    free+=cache;
    printf(“k(%d), right index(%d), free size(%d)\n\n”, k, right, cache);
  }
  return free;
```

It find 2 pairs: 1. Next block containing p and its buddy. 2. Current block containing bd\_end and its buddy. For each pair, if either of them is allocated, move the free one to size k's free list if that free block is within free range.

**The thing surprised me is if we add up all the size of pushed-to-free-list blocks, the total is exactly the available size \(bd\_end - p\)!** **This mechanisms significantly reduced the first multiple split!**

I re-write code and prove the above is true. By set nsize to 5, we have:

```c
bd: heap bd_init k 0, num of block: 16, sze: 1
bd: heap bd_init k 1, num of block: 8, sze: 1
bd: heap bd_init k 2, num of block: 4, sze: 1
bd: heap bd_init k 3, num of block: 2, sze: 1
bd: heap bd_init k 4, num of block: 1, sze: 1
using 9, round up 16
bd: 16 meta bytes for managing 256 bytes of memory
Bump n 0 for k(1)
Bump n 0 for k(2)
Bump n 0 for k(3)
Bump n 0 for k(4)
k(0), left index(1), free size(16)
k(0) is skipped for index(16)!
k(0), right index(16), free size(0)

Bump n 0 for k(1)
k(1), left index(1), free size(32)
k(1) is skipped for index(8)!
k(1), right index(8), free size(0)

Bump n 0 for k(2)
k(2), left index(1), free size(64)
k(2) is skipped for index(4)!
k(2), right index(4), free size(0)

Bump n 0 for k(3)
k(3), left index(1), free size(128)
k(3) is skipped for index(2)!
k(3), right index(2), free size(0)

used 16. size of free: 240, expected 240
=== buddy allocator state ===
size 0 (16): free list:  0x103f56010
  alloc: 1 0 0 0 0 0 0 0
size 1 (32): free list:  0x103f56020
  alloc: 1 0 0 0
  split: 1 0 0 0 0 0 0 0
size 2 (64): free list:  0x103f56040
  alloc: 1 0
  split: 1 0 0 0
size 3 (128): free list:  0x103f56080
  alloc: 1
  split: 1 0
size 4 (256): free list:
  alloc: 1
  split: 1
```

The key idea is:

* Round up the usage.
* Loop each size: 
* For left pair, and right pair.  Any xor is 1 \(either buddy or me is allocated\), we find the free one, and put into its size of free list.
* If xor is 0, do nothing for current size of free list.

**How we determine which one from pair is free?** _Check whether it is within range\(p, bd\_end\)!_

From the sample above, the meta data size was 9, round up to LEAF\_SIZE \(16 bytes\), and free memory range starts from there. In above case, size 0 got 16, size 2 got 32, 64, 128. Total is 240.

### Summary

In other words, the concept is very similar to divide free memory to all free lists. Let’s assume p points to the end of first 16 bytes. Free memory range is \[p, end\).  After allocator initialization, max size free list is empty, So we have: size 4: 0, size 3: 64, size 2: 32,  size 1: 16.

**The reason the memory area is 2^x is the address of each buddies are only different by 1 bit.** So it is easy to find.

