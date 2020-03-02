---
description: Re-designing code to increase parallelism
---

# Lab 7 Lock

## Introduction

You’ll gain experience in re-designing code to increase parallelism. A common symptom of poor parallelism on multi-core machines is high lock contention. Improving parallelism often involves changing both data structures and locking strategies in order to reduce contention. You’ll do this for the xv6 memory allocator and block cache. 

[Lab: locks](https://pdos.csail.mit.edu/6.828/2019/labs/lock.html)

## Task 1: Memory Allocator

The program user/kalloctest stresses xv6’s memory allocator: three processes grow and shrink their address spaces, resulting in many calls to kalloc and kfree. kalloc and kfree obtain kmem.lock. kalloctest prints the number of test-and-sets that did not succeed in acquiring the kmem lock \(and some other locks\), which is a rough measure of contention:

```c
$ **kalloctest**
start test0
test0 results:
=== lock kmem/bcache stats
lock: kmem: #test-and-set 161724 #acquire() 433008
lock: bcache: #test-and-set 0 #acquire() 812
=== top 5 contended locks:
lock: kmem: #test-and-set 161724 #acquire() 433008
lock: proc: #test-and-set 290 #acquire() 961
lock: proc: #test-and-set 240 #acquire() 962
lock: proc: #test-and-set 72 #acquire() 907
lock: proc: #test-and-set 68 #acquire() 907
test0 FAIL
start test1
total allocated number of pages: 200000 (out of 32768)
test1 OK
```

The root cause of lock contention in kalloctest is that kalloc\(\) has a single free list, protected by a single lock.

### Your job

To remove lock contention, you will have to redesign the memory allocator to avoid a single lock and list. The basic idea is to maintain a free list per CPU, each list with its own lock.

### Solution

#### In `kalloc`

```c
// Allocate one 4096-byte page of physical memory.
// Returns a pointer that the kernel can use.
// Returns 0 if the memory cannot be allocated.
void *
kalloc(void)
{
  push_off();
  int id = cpuid();

  struct run *r;

  acquire(&mem_cpus[id].lock);

  r = mem_cpus[id].freelist;
  if(r) {
    mem_cpus[id].freelist = r->next;
  } 
  // We have to release the lock here
  // if we hold the lock and try to steal other cpu’s freelist,
  // we will get deadlock!
  // ex: CPU a holds lock A, to grab lock B,
  // CPU 2 holds B, try to grab lock A.
  release(&mem_cpus[id].lock);

  if (!r) {
    // We need to borrow from other cpu’s free list.
    for (int I =0; I < NCPU; I++) {
      if (I == id) {
        continue;
      }
      acquire(&mem_cpus[I].lock);
      struct run *f = mem_cpus[I].freelist;
      if (f) {
        r = f;
        mem_cpus[I].freelist = f->next;
      }
      release(&mem_cpus[I].lock);
      if (r)
        break;
    }
  }

  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk

  pop_off();
  return (void*)r;
}
```

1. Try to get free memory from process own free list.
2. Steal free memory from other process’ free list.
3. Use lock when access free list.
4. Don’t hold more than one lock. Avoid deadlock.

## Task 2: Buffer Cache

### The Problem

If multiple processes use the file system intensively, they will likely contend for `bcache.lock`, which protects the disk block cache in `kernel/bio.c`. `bcachetest` creates several processes that repeatedly read different files in order to generate contention on `bcache.lock`.

### Your Task

Modify the block cache so that the number of test-and-sets for all locks in the bcache is close to zero when running bcachetest. Ideally the sum of test-and-sets of all locks involved in the block cache should be zero, but it’s OK if the sum is less than 500. Modify bget and brelse so that concurrent lookups and releases for different blocks that are in the bcache are unlikely to conflict on locks \(e.g., don’t all have to wait for bcache.lock\). You must maintain the invariant that at most one copy of a block is cached.

### Solution

Instead of using linked list, look up block numbers in the cache with a **hash table that has a lock per hash bucket.**

#### Implement hashable

```c
#define HASH_TABLE_SIZE 503

struct ht_bucket {
  uint blockno;
  struct buf buf; // TODO: make it to linked list of buf*
  struct spinlock bucket_lock;
};

struct {
  int size;
  int count;
  struct spinlock lock;
  struct ht_bucket buckets[HASH_TABLE_SIZE]; // array of pointers to ht_item*
} bcache_ht;

void ht_init();
void ht_insert(uint bno, struct buf buffer);
struct buf* ht_find(uint bno);
void ht_delete(uint bno);
uint ht_hash(uint bno);

void ht_init()
{
  initlock(&bcache_ht.lock, "bcache");
  for (int i = 0; i < HASH_TABLE_SIZE; i++) {
    initsleeplock(&bcache_ht.buckets[i].buf.lock, "buffer"); 
  }
}

uint ht_hash(uint bno)
{
  return bno % HASH_TABLE_SIZE;
}

struct buf* ht_find(uint bno)
{
  uint hash = ht_hash(bno);
  struct buf *b = &bcache_ht.buckets[hash].buf;
  if (b->refcnt > 0)
  {
    return b;
  }
  return 0;
}

void ht_insert(uint bno, struct buf buffer)
{
  uint hash = ht_hash(bno);
  struct buf *b = ht_find(bno);
  if (b && b->dev != buffer.dev) {
    printf("Found collision item block num:%d. Hash(%d)\n", bno, hash);
  }
  bcache_ht.buckets[hash].buf = buffer;
}

void ht_print()
{
  for (int i = 0; i < HASH_TABLE_SIZE; i++) {
    struct buf* b = &bcache_ht.buckets[i].buf;
    if (b && b->refcnt > 0) {
      printf("Buffer(%d) is valid. Hash(%d)\n", b->blockno, i);
    }
  }
}
```

#### Modify `bget` to use hash table

```c
static struct buf*
bget(uint dev, uint blockno)
{
  uint hash = ht_hash(blockno);

  acquire(&bcache_ht.buckets[hash].bucket_lock);

  struct buf *b = ht_find(blockno);
  if (b && b->dev == dev) {
    b->refcnt++;
    release(&bcache_ht.buckets[hash].bucket_lock);
    acquiresleep(&b->lock);
    return b;
  }

  struct buf* new_buf = &bcache_ht.buckets[hash].buf;
  new_buf->dev = dev;
  new_buf->blockno = blockno;
  new_buf->valid = 0;
  new_buf->refcnt = 1;
  ht_insert(blockno, *new_buf);
  release(&bcache_ht.buckets[hash].bucket_lock);
  acquiresleep(&new_buf->lock);
  return new_buf;
}
```

#### `brelse` doesn’t need to acquire the `bcache lock`

```c
// Release a locked buffer.
void brelse(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic(“brelse”);

  releasesleep(&b->lock);

  acquire(&bcache_ht.lock);
  b->refcnt—;
  release(&bcache_ht.lock);
}
```

## Thoughts

Lock contention is a hard to solve problem. My buffer cache solution is a little bit hack, since the actual optimal solution is to have a concurrent hash table, each bucket is a linked list. Also another list to track least significant use. Similar to LRU cache.

**Result**

All tests except `writebig` and `bigdir` tests have passed.

