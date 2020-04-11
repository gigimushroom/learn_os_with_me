# XV6 VS Real World

### Buffer Cache

Buffer cache serves the same 2 purposes as real-world ones: 1. Caching 2. Synchronizing access to the disk

Xv6 uses a linked list, and LRU. Modern buffer caches uses hash table for lookups and a heap for LUR evictions. Modern buffer caches are integrated with virtual memory system to support memory-mapped files.

### Logging

#### What xv6 is doing

In xv6, A commit cannot occur concurrently with file-system system calls. In other words, if log is committing, other system calls are blocked and wait.

In xv6, logs recorded the entire block, even if only one byte is changed.

It performs synchronous log writes, a block at a time.

#### Real world

Real logging system address all of above issues.

### On-disk layout

The same with xv6 and real world.

The most inefficient part of the file system layout is the **directory**, which requires a linear scan over all the disk blocks during each lookup.

Real world uses on-disk balanced tree of blocks.

### Separating disk management from file system

Hard. Xv6 does not support combining many disks into a single logical disk.

### Other xv6 file system missing pieces

Lack of snapshot, and incremental backup.

