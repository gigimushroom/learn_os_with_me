# Design a log system for crash recovery

### Crash in the file system can lead to inconsistent state.

For example: Depending on the order of disk writes, crash may leave an inode with a reference to a content block that is marked free. Or leave an allocated but not yet referenced content block.

### Log Header

The header block contains an array of sector numbers, one for each of the logged blocks, and the count of log blocks. The count is either 0, indicating there is no transaction. Or non-zero, indicating the log contains a complete committed transactions with a number of logged blocks.

XV6 writes the header block when a txn commits, but not before. So it is atomic.

### Group Commit

Committing several txns together. Logging system only commits when no fs system calls ongoing.

### Log Disk Space

XV6 dedicates a fixed amount of space on disk to hold log. The total number of blocks written by the system calls must fit in the space. 1. No single system call can be allowed to write more distinct blocks than allowed. 2. A system call must wait until there is enough space in log then begins.

#### Special Case: write

XV6’s write system call breaks up large writes into multiple smaller writes that fit in the log.

## Data structure

Code in `log.c`

```c
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

## Pattern for using the log in system call

```c
begin_op();
ilock(f->ip);
r = writei(f->ip, …);
iunlock(f->ip);
end_op();
```

### `begin_op`

Wait and sleep if the log is currently committing. Sleep when log is reaching MAX allowed size. Otherwise, increment the outstanding system call counter in log metadata, and continue.

```c
// called at the start of each FS system call.
void
begin_op(int dev)
{
  acquire(&log[dev].lock);
  while(1){
    if(log[dev].committing){
      sleep(&log, &log[dev].lock);
    } else if(log[dev].lh.n + (log[dev].outstanding+1)*MAXOPBLOCKS > LOGSIZE){
      // this op might exhaust log space; wait for commit.
      sleep(&log, &log[dev].lock);
    } else {
      log[dev].outstanding += 1;
      release(&log[dev].lock);
      break;
    }
  }
}
```

### `ilock`

Lock the given inode. Reads the inode from disk if necessary.

```c
void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic("ilock");

  acquiresleep(&ip->lock);

  if(ip->valid == 0){
    bp = bread(ip->dev, IBLOCK(ip->inum, sb));
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```

This part is very important, since the inode we have in-memory might not load the latest content from disk!

### `iunlock`

Unlock the given inode by releasing the sleep lock.

```c
void
iunlock(struct inode *ip)
{
  if(ip == 0 || !holdingsleep(&ip->lock) || ip->ref < 1)
    panic("iunlock");

  releasesleep(&ip->lock);
}
```

### `end_op`

Called at the end of FS system call. Commits if this was the last outstanding operation.

```c
void
end_op(int dev)
{
  int do_commit = 0;

  acquire(&log[dev].lock);
  log[dev].outstanding -= 1;
  if(log[dev].committing)
    panic(“log[dev].committing”);
  if(log[dev].outstanding == 0){
    do_commit = 1;
    log[dev].committing = 1;
  } else {
    // begin_op() may be waiting for log space,
    // and decrementing log[dev].outstanding has decreased
    // the amount of reserved space.
    wakeup(&log);
  }
  release(&log[dev].lock);

  if(do_commit){
    // call commit w/o holding locks, since not allowed
    // to sleep with locks.
    commit(dev);
    acquire(&log[dev].lock);
    log[dev].committing = 0;
    wakeup(&log);
    release(&log[dev].lock);
  }
}
```

### Fun Part

A\) Linux ext2 -&gt; ext3 is using the logging system. See [journaling the linux ext2fs filesystem](http://e2fsprogs.sourceforge.net/journal-design.pdf) 

B\) Logging is similar to DB’s logging and transaction design.

