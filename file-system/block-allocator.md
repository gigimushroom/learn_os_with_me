# Block Allocator

Xv6’s block allocator maintains a free bitmap on disk, with on bit per block.

### Allocate a disk block

Find the first available block, mark as in-use.

```c
#define BSIZE 1024 // block size
```

```c
// Block containing inode I
#define IBLOCK(I, sb)     ((i) / IPB + sb.inodestart)

// Bitmap bits per block
#define BPB           (BSIZE*8)

// Block of free map containing bit for block b
#define BBLOCK(b, sb) ((b)/BPB + sb.bmapstart)
```

Note: `sb` is superblock. `struct superblock sb;`

```c
static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;

  bp = 0;
  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}
```

The outer loop reads each block of bitmap bits. The inner loop checks all `BPB` bits in a single bitmap block.

### Free a disk block

Find it and mark the bit as 0.

```c
static void
bfree(int dev, uint b)
{
  struct buf *bp;
  int bi, m;

  bp = bread(dev, BBLOCK(b, sb));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic(“freeing free block”);
  bp->data[bi/8] &= ~m;
  log_write(bp);
  brelse(bp);
}
```

### Both allocation and free must be called inside a transaction

