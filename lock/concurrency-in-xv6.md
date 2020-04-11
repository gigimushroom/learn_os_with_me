# Concurrency in Xv6

## Locking patterns

### Pattern 1: Design locked data structure

One lock for the set of items, plus one lock per item.

#### Example: The filesystem’s block cache.

Each cached block is stored in a `struct buf` \(kernel/buf.h:1\). A `struct buf` has a lock field which helps ensure that only one process uses a given disk block at a time.

When checking if a block is cached, or change the set of cached blocks, must hold the `bcache.lock`; after the code founds the needed `struct buf`, it can release `bcache.lock`.

### Pattern 2: Design locks to run in different threads

A lock is acquired at the start of a sequence that must appear atomic, and released when that sequence ends. If the sequence starts and ends in different functions, or different threads, or on different CPUs, then the lock acquire and release must do the same.

#### Example

`acquire` process’ lock in `yield` , but release in the scheduler thread rather than the acquiring process.

Another example is the `acquiresleep` in `ilock` \(kernel/fs.c:290\); this code often sleeps while reading the disk; it may wake up on a different CPU, which means the lock may be acquired and released on different CPUs.

```c
// Lock the given inode.
// Reads the inode from disk if necessary.
void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic(“ilock”);

  acquiresleep(&ip->lock);
…
```

### Pattern 3: How to free an object

#### Problem

Freeing an object that is protected by a lock is hard. If other thread is waiting in `acquire` to use the object; freeing the object when unlock will cause the waiting thread to malfunction.

#### Solution

Track how many references to the object exist.

#### Example

See `pipeclose` \(kernel/pipe.c:59\) for an example; `pi->readopen` and `pi->writeopen` track whether the pipe has file descriptors referring to it.

```c
void
pipeclose(struct pipe *pi, int writable)
{
  acquire(&pi->lock);
  if(writable){
    pi->writeopen = 0;
    wakeup(&pi->nread);
  } else {
    pi->readopen = 0;
    wakeup(&pi->nwrite);
  }
  if(pi->readopen == 0 && pi->writeopen == 0){
    release(&pi->lock);
    kfree((char*)pi);
  } else
    release(&pi->lock);
}
```

## Lock-like patterns

A lock protects the flag or reference count, it is the latter that prevents the object from being prematurely freed.

### Example: Filesystem looks up path

`namex` holds lock for each directory, release it before acquiring the next directory. Otherwise, it might deadlock itself when accessing `a/./b`.

```c
static struct inode*
namex(char *path, int nameiparent, char *name)
{
  struct inode *ip, *next;

  if(*path == ‘/‘)
    ip = iget(ROOTDEV, ROOTINO);
  else
    ip = idup(myproc()->cwd);

  while((path = skipelem(path, name)) != 0){
    ilock(ip);
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    if(nameiparent && *path == ‘\0’){
      // Stop one level early.
      iunlock(ip);
      return ip;
    }
    if((next = dirlookup(ip, name, 0)) == 0){
      iunlockput(ip);
      return 0;
    }
    iunlockput(ip);
    ip = next;
  }
  if(nameiparent){
    iput(ip);
    return 0;
  }
  return ip;
}
```

#### Solution

The solution is for the loop to carry the directory `inode` over to the next iteration with its reference count incremented, but not locked.

**Explain**

`next = dirlookup(ip, name, 0)` calls `iget`, which increment ref count. Each iteration only release current `inode`, and set next `inode` pointer, but not acquiring next directory lock.

### Other lock-like patterns

#### Protect by implicit structure rather than locks

A data object may be protected from concurrency in different ways at different points in its lifetime, and the protection may take the form of implicit structure rather than explicit locks.

For example, when a physical page is free, it is protected by `kmem.lock` \(kernel/kalloc.c:24\). If the page is then allocated as a pipe \(kernel/pipe.c:23\), it is protected by a different lock \(the embedded `pi->lock`\). If the page is re-allocated for a new process’s user memory, it is not protected by a lock at all.

#### Disable interrupts

Disabling interrupts causes the calling code to be atomic with respect to timer interrupts that could force a context switch, and thus move the process to a different CPU.

## No locks at all

A few places in xv6 share mutable data with no locks at all.

The `started` variable in main.c \(kernel/main.c:7\), used to prevent other CPUs from running until CPU zero has finished initializing xv6; the `volatile` ensures that the compiler actually generates load and store instructions.

```cpp
volatile static int started = 0;

// start() jumps here in supervisor mode on all CPUs.
void
main()
{
  if(cpuid() == 0){
    consoleinit();
    …
    userinit();      // first user process
    __sync_synchronize();
    started = 1;
  } else {
    while(started == 0)
      ;
    __sync_synchronize();
    printf(“hart %d starting\n”, cpuid());
    …
  }
```

## Study Topics

### c `volatile` keyword

\[volatile \(computer programming\) - Wikipedia\]\([https://en.wikipedia.org/wiki/Volatile\_\(computer\_programming](https://en.wikipedia.org/wiki/Volatile_%28computer_programming)\)\)

The **volatile** \[keyword\]\([https://en.wikipedia.org/wiki/Keyword\_\(computer\_programming](https://en.wikipedia.org/wiki/Keyword_%28computer_programming)\)\) indicates that a \[value\]\([https://en.wikipedia.org/wiki/Value\_\(computer\_science](https://en.wikipedia.org/wiki/Value_%28computer_science)\)\) may change between different accesses, even if it does not appear to be modified. This keyword prevents an [optimizing compiler](https://en.wikipedia.org/wiki/Optimizing_compiler) from optimizing away subsequent reads or writes and thus incorrectly reusing a stale value or omitting writes. Volatile values primarily arise in hardware access \( [memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O) \), where reading from or writing to memory is used to communicate with [peripheral devices](https://en.wikipedia.org/wiki/Peripheral_device) , and in \[threading\]\([https://en.wikipedia.org/wiki/Thread\_\(computing](https://en.wikipedia.org/wiki/Thread_%28computing)\)\) , where a different thread may have modified a value.

There are [memory barrier](https://en.wikipedia.org/wiki/Memory_barrier) operations available on platforms \(which are exposed in C++11\) that should be preferred instead of volatile .

In C\#, When a field is marked volatile, the compiler is instructed to generate a “memory barrier” or “fence” around it, which prevents instruction reordering or caching tied to the field.

In XV6, the volatile ensures that the compiler actually generates load and store instructions. Together with `__sync_synchronize`, XV6 ensures `started` is modified by CPU 0, and shares to other CPUs with no memory re-order. A similar function is `__sync_fetch_and_add`.

## Parallelism

Locking is primarily about suppressing parallelism in the interests of correctness. Because performance is also important, kernel designers often have to think about how to use locks in a way that achieves both correctness and good parallelism.



