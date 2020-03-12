# XV6 File System

![](../.gitbook/assets/image%20%287%29.png)

### Buffer Cache

1. Sync access to disk blocks to ensure only one copy of a block in memory and only one kernel thread can use this copy.
2. Cache popular blocks See `bio.c` **Source code**

   ```text
   struct buf {
   int valid;   // has data been read from disk?
   int disk;    // does disk “own” buf?
   uint dev;
   uint blockno;
   struct sleeplock lock;
   uint refcnt;
   struct buf *prev; // LRU cache list
   struct buf *next;
   uchar data[BSIZE];
   };
   ```

   ```text
   struct {
   struct spinlock lock;
   struct buf buf[NBUF];

   // Linked list of all buffers, through prev/next.
   // head.next is most recently used.
   struct buf head;
   } bcache;
   ```

### Logging

#### Overview

Designed for crash recovery. Every write is logged to log disk. After commit, all data write to actual disk block. Code in `log.c` **Log struct**

```text
// Contents of the header block, used for both the on-disk header block
// and to keep track in memory of logged block# before commit.
struct logheader {
  int n; //xiaying: number of current added block.
  int block[LOGSIZE];
};

struct log {
  struct spinlock lock;
  int start;
  int size; //xiaying: total allowed capacity.
  int outstanding; // how many FS sys calls are executing.
  int committing;  // in commit(), please wait.
  int dev;
  struct logheader lh;
};
struct log log[NDISK];

static void recover_from_log(int);
static void commit(int);
```

#### key features

**Each write to disk system use transaction:**

```text
begin_op(); 
ilock(f->ip); 
r = writei(f->ip, …); 
iunlock(f->ip); 
end_op();
```

#### Use group commits

`end_op` commits if this was the last outstanding operation.

#### Restart file system trigger recovering from log.

1. Read log head from disk.
2. Copy committed blocks from log to their home location.
3. Clear the log head since we are done recovering.

   ```text
   // Copy committed blocks from log to their home location
   static void
   install_trans(int dev)
   {
   int tail;

   for (tail = 0; tail < log[dev].lh.n; tail++) {
    struct buf *lbuf = bread(dev, log[dev].start+tail+1); // read log block
    struct buf *dbuf = bread(dev, log[dev].lh.block[tail]); // read dst
    memmove(dbuf->data, lbuf->data, BSIZE);  // copy block to dst
    bwrite(dbuf);  // write dst to disk
    bunpin(dbuf); // xiaying: why we pin 
    brelse(lbuf);
    brelse(dbuf);
   }
   }
   ```

   ```text
   static void
   recover_from_log(int dev)
   {
   read_head(dev);
   install_trans(dev); // if committed, copy from log to disk
   log[dev].lh.n = 0;
   write_head(dev); // clear the log
   }
   ```

   **Fun Part**

   A\) Linux ext2 -&gt; ext3 is using the logging system. See [journaling the linux ext2fs filesystem](http://e2fsprogs.sourceforge.net/journal-design.pdf) B\) Logging is similar to DB’s logging and transaction design.

### Inode Layer

#### Overview

The term node can refer 2 things: 1. On disk data structure contains file’s size and list of data blocks numbers. 2. Refer to in-memory node, contains a copy of the on-disk node and extra info. 

![](../.gitbook/assets/image%20%289%29.png)

```text
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

```text
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

See `fs.c`

#### Inode Key Design \(From book\)

**An inode describes a single unnamed file.**

The inode disk structure holds metadata: the file's type, its size, the number of links referring to it, and the list of blocks holding the file's content.

The inodes are laid out sequentially on disk at sb.startinode. Each inode has a number, indicating its position on the disk.

The kernel keeps a cache of in-use inodes in memory to provide a place for synchronizing access to inodes used by multiple processes. The cached inodes include book-keeping information that is not stored on disk: ip-&gt;ref and ip-&gt;valid.

An inode and its in-memory representation go through a sequence of states before they can be used by the rest of the file system code.

* **Allocation**: an inode is allocated if its type \(on disk\)

  is non-zero. ialloc\(\) allocates, and iput\(\) frees if

  the reference and link counts have fallen to zero.

* **Referencing in cache**: an entry in the inode cache

  is free if ip-&gt;ref is zero. Otherwise ip-&gt;ref tracks

  the number of in-memory pointers to the entry \(open

  files and current directories\). iget\(\) finds or

  creates a cache entry and increments its ref; iput\(\)

  decrements ref.

* **Valid**: the information \(type, size, &c\) in an inode

  cache entry is only correct when ip-&gt;valid is 1.

  ilock\(\) reads the inode from

  the disk and sets ip-&gt;valid, while iput\(\) clears

  ip-&gt;valid if ip-&gt;ref has fallen to zero.

* **Locked**: file system code may only examine and modify

  the information in an inode and its content if it

  has first locked the inode.

  Thus a typical sequence is:

  ```text
  ip = iget(dev, inum)
  ilock(ip)
  ... examine and modify ip->xxx ...
  iunlock(ip)
  iput(ip)
  ```

ilock\(\) is separate from iget\(\) so that system calls can get a long-term reference to an inode \(as for an open file\) and only lock it for short periods \(e.g., in read\(\)\).

The separation also helps avoid deadlock and races during pathname lookup. iget\(\) increments ip-&gt;ref so that the inode stays cached and pointers to it remain valid.

Many internal file system functions expect the caller to have locked the inodes involved; this lets callers create multi-step atomic operations.

4 Locks in inode code. The icache.lock spin-lock protects the allocation of icache entries. Since ip-&gt;ref indicates whether an entry is free, and ip-&gt;dev and ip-&gt;inum indicate which i-node an entry holds, one must hold icache.lock while using any of those fields.

An ip-&gt;lock sleep-lock protects all ip-&gt; fields other than ref, dev, and inum.  
One must hold ip-&gt;lock in order to

### read or write that inode's ip-&gt;valid, ip-&gt;size, ip-&gt;type, &c.

#### Design Decisions

**Lock design**

Lock inode when reading from disk \(buffer cache\) Not locking when reading inode from memory copy. To avoid: deadlock on multiple scenarios \(ex: looking up current dir.\) See `namex` or pathname section in xv6 book.

**Look up path name**

Lock the directory it current visit, so a separate lookup can operate concurrently. It also solves one thread unlink dir A, but another thread is accessing dir A. The solution in xv6 is: access will bump the ref counter, so even link\_num is 0, the inode will not be freed. So entire access is safe.

**Link and Ref Count**

Link and ref count are 2 things on node. If number of links is 0, ref count is &gt; 0. It means we don’t have any directory link to the file, but some process are still accessing the node, we cannot delete them.

### Directory Layers

Its inode has type T\_DIR and its data is a sequence of directory entries. Each entry is a struct dirent \(kernel/fs.h:56\), which contains a name and an inode number.

### File Descriptor layers

Everything is file. Each process has list of opened files. There is a global file tables contains all the opened files.

### File has interface to alloc, close, stat, read, write. Operates based on file type: pipe, device, inode.

## Lab 8 File System

### Symlink Lab

The key is _where we store the target path._ We should store it in `inode`’s data blocks. By using `readi` and `writei` , we could read and write into/from the `inode->data[0]’s block`

Bad decision I made: I originally tried to add a new char array in inode struct, but it does not use buffer cache, not sync to `dinode`, not logged properly, and not store back to disk properly.

**sys\_open**

```text
if ((ip->type == T_SYMLINK) && !(omode & O_NOFOLLOW)){
    int count = 0;
    while (ip->type == T_SYMLINK && count < 10) {
      int len = 0;
      readi(ip, 0, (uint64)&len, 0, sizeof(int));

      if(len > MAXPATH)
        panic(“open: corrupted symlink inode”);

      readi(ip, 0, (uint64)path, sizeof(int), len + 1);
      iunlockput(ip);

      *//printf(“OMG!!!!%s\n”, path);*
      if((ip = namei(path)) == 0){
        end_op(ROOTDEV);
        return -1;
      }
      ilock(ip);
      count++;
    }
    if (count >= 10) {
      printf(“We got a cycle!\n”);
      iunlockput(ip);
      end_op(ROOTDEV);
      return -1;
    }
  }
```

#### New System Call: sys\_symlink

```text
uint64
sys_symlink(void)
{
  char target[MAXPATH], path[MAXARG];
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0){
    return -1;
  }
  *//printf(“creating a sym link. Target(%s). Path(%s)\n”, target, path);*

  begin_op(ROOTDEV);
  struct inode *ip = create(path, T_SYMLINK, 0, 0);
  if(ip == 0){
    end_op(ROOTDEV);
    return -1;
  }

  int len = strlen(target);
  writei(ip, 0, (uint64)&len, 0, sizeof(int));
  writei(ip, 0, (uint64)target, sizeof(int), len + 1);
  iupdate(ip);
  iunlockput(ip);

  end_op(ROOTDEV);
  return 0;
}
```

A bit more on `readi` and `writei`

```text
*// Read data from inode.*
*// Caller must hold ip->lock.*
*// If user_dst==1, then dst is a user virtual address;*
*// otherwise, dst is a kernel address.*
int
readi(struct inode *ip, int user_dst, uint64 dst, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > ip->size)
    n = ip->size - off;

  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    if(either_copyout(user_dst, dst, bp->data + (off % BSIZE), m) == -1) {
      brelse(bp);
      break;
    }
    brelse(bp);
  }
  return n;
}
```

bmap\(ip, off/BSIZE\) returns the disk block address of the nth block in inode ip. `readi` starts from offset, get the buffer containing the data, copy data from buffer’s data to the destination address. `writei` works similar. Write data from src to inode

**How `bmap` works**

```text
*// Inode content*
*//*
*// The content (data) associated with each inode is stored*
*// in blocks on the disk. The first NDIRECT block numbers*
*// are listed in ip->addrs[].  The next NINDIRECT blocks are*
*// listed in block ip->addrs[NDIRECT].*

*// Return the disk block address of the nth block in inode ip.*
*// If there is no such block, bmap allocates one.*
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    *// Load indirect block, allocating if necessary.*
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
```

**How `bread` works**

```text
*// Return a locked buf with the contents of the indicated block.*
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if(!b->valid) {
    virtio_disk_rw(b->dev, b, 0);
    b->valid = 1;
  }
  return b;
}
```

**Thank you, file system!**

