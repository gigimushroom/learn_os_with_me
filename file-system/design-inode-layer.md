# Design Inode Layer

 Inode has 2 related meanings: 

1. On-disk data structure containing a file’s size and list of data block numbers. 

2. In-memory node, which contains a copy of the on-disk inode as well as extra info needed by kernel.

```c
// On-disk inode structure
struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEVICE only)
  short minor;          // Minor device number (T_DEVICE only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

```c
// in-memory copy of an inode
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?
  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

### On-disk `inode`

The on-disk inodes are packed into a contiguous area of disk. Every one is same size. `struct dinode`

`type` could be file, directory, device, or 0 \(means on-disk node is empty\). `nlink` counts the number of directory entries that refer to this node.

### In-memory `inode`

`struct inode`

The kernel stores an inode in memory only if there are C pointers referring to that node.

The `ref` field counts the number of C pointers referring to the in-memory inode. Kernel discards the inode from memory if reference count drops to 0.

The `iget` and `iput` functions acquire and release pointers to an inode, modifying the reference count.

### Locking in `inode` \(4 kinds\)

#### Big lock on in-memory inodes array

`icache.lock` protects the invariant that an inode is present in the cache at most once.

#### Lock on each in-memory inode

Each in-memory inode has a lock field containing a sleep lock, which ensures exclusive access to inode’s important data.

#### Inode reference count

If a `ref` on inode is &gt; 0, system will maintain the inode in the cache, and the cache entry will not be re-used.

#### Number of links

Each inode contains `link` field \(stored in disk, copied in memory\), counts the number of directory entries that refer to a file. Xv6 won't free an inode if its link count is greater than 0.

### Separate reference count and locking

Separating acquisition of inode pointers from locking helps avoid deadlock. Multiple processes can hold a C pointer to an inode returned by iget, but only one process can lock the inode at a time.

`iget` only increments reference count. `ilock` acquires the inode’s lock.

### Why we have in-memory cache of inode?

Main job is to providing synchronizing access by multiple process. Only one process can access the inode at a time. Second is caching. If an inode is frequently used, the buffer cache might keep it at the beginning of its chain if it isn't kept by inode cache.

The inode cache is _write-through_. Any code modifies the cached inode must immediately write it to disk \(`iupdate`\).

## Implement Inodes

### Allocate a new inode

Similar to `balloc` for allocating a disk block. It loops over the inode structures on disk, one at a time, locking for one that is marked free. When finds one, it claims by writing the new `type`. Return an entry from the in-memory inode cache. \(Calling `iget`\)

```c
// Allocate an inode on device dev.
// Mark it as allocated by  giving it type type.
// Returns an unlocked but allocated and referenced inode.
struct inode*
ialloc(uint dev, short type)
{
  int inum;
  struct buf *bp;
  struct dinode *dip;

  for(inum = 1; inum < sb.ninodes; inum++){
    bp = bread(dev, IBLOCK(inum, sb));
    dip = (struct dinode*)bp->data + inum%IPB;
    if(dip->type == 0){  // a free inode
      memset(dip, 0, sizeof(*dip));
      dip->type = type;
      log_write(bp);   // mark it allocated on the disk
      brelse(bp);
      return iget(dev, inum);
    }
    brelse(bp);
  }
  panic("ialloc: no inodes");
}
```

### Get inode from cache

`iget` looks thought the in-memory inode cache for an active entry. Bump the ref count and return. Or claim a new empty one.

```c
// Find the inode with number inum on device dev
// and return the in-memory copy. Does not lock
// the inode and does not read it from disk.
static struct inode*
iget(uint dev, uint inum)
{
  struct inode *ip, *empty;

  acquire(&icache.lock);

  // Is the inode already cached?
  empty = 0;
  for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++){
    if(ip->ref > 0 && ip->dev == dev && ip->inum == inum){
      ip->ref++;
      release(&icache.lock);
      return ip;
    }
    if(empty == 0 && ip->ref == 0)    // Remember empty slot.
      empty = ip;
  }

  // Recycle an inode cache entry.
  if(empty == 0)
    panic("iget: no inodes");

  ip = empty;
  ip->dev = dev;
  ip->inum = inum;
  ip->ref = 1;
  ip->valid = 0;
  release(&icache.lock);

  return ip;
}
```

### Release a reference count on in-memory inode

Drop the reference count for an in-memory inode. If no more reference, and number of directory links is 0, free the inode on disk.

```c
// Drop a reference to an in-memory inode.
// If that was the last reference, the inode cache entry can
// be recycled.
// If that was the last reference and the inode has no links
// to it, free the inode (and its content) on disk.
// All calls to iput() must be inside a transaction in
// case it has to free the inode.
void
iput(struct inode *ip)
{
  acquire(&icache.lock);

  if(ip->ref == 1 && ip->valid && ip->nlink == 0){
    // inode has no links and no other references: truncate and free.

    // ip->ref == 1 means no other process can have ip locked,
    // so this acquiresleep() won’t block (or deadlock).
    acquiresleep(&ip->lock);

    release(&icache.lock);

    itrunc(ip);
    ip->type = 0;
    iupdate(ip);
    ip->valid = 0;

    releasesleep(&ip->lock);

    acquire(&icache.lock);
  }

  ip->ref—;
  release(&icache.lock);
}
```

### Locking when freeing inode

When `nlink` is 0, the inode is not trace-able anymore, so not other process can use path to locate this inode. We do not need to worry just before truncating, some other process grab and add more content to it.

### Every FS system calls can write to disk

`iput` can write to disk, when inode should be freed. So even a `read` could write to disk. We must wrap ALL system calls into transactions if they use the file system.

### Handle orphan inodes

There is a challenging interaction between `iput()` and crashes.

`iput()` doesn’t truncate a file immediately when the link count for the file drops to zero \(it tries to acquire the lock first\), because some process might still hold a reference to the inode in memory: A process might still be reading and writing to the file, because it successfully opened it.

If crash happened in the meantime \(before `fd` is closed\), the file will be marked allocated on disk still but no directory entry will point to it.

#### How to clean those un-linked inode?

Solution 1: On recovery, scans the whole file system, for files that are marked as allocated, but no directory entry pointing to them. Free them.

Solution 2: Track those 'link count drops to 0 but still has reference count' files on disk. On recovery, scan the list, and free them.

