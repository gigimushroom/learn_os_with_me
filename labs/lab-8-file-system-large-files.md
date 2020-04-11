---
description: Increase the maximum size of an xv6 file.
---

# Lab 8 File System: Large Files

## The Problem 

Currently xv6 files are limited to 268 blocks, or 268\*BSIZE bytes \(BSIZE is 1024 in xv6\). This limit comes from the fact that an xv6 inode contains 12 “direct” block numbers and one “singly-indirect” block number, which refers to a block that holds up to 256 more block numbers, for a total of 12+256=268 blocks.

![](../.gitbook/assets/image%20%2812%29.png)

## Your Task

### High Level Explain

Increase the maximum size of an xv6 file. You’ll change the xv6 file system code to support a "doubly-indirect” block in each inode, containing 256 addresses of singly-indirect blocks, each of which can contain up to 256 addresses of data blocks. The result will be that a file will be able to consist of up to 256\*256+256+11 blocks \(11 instead of 12, because we will sacrifice one of the direct block numbers for the double-indirect block\).

### Implementation Details

Modify `bmap()` so that it implements a doubly-indirect block, in addition to direct blocks and a singly-indirect block. You’ll have to have only 11 direct blocks, rather than 12, to make room for your new doubly-indirect block; you’re not allowed to change the size of an on-disk inode. The first 11 elements of `ip->addrs[]` should be direct blocks; the 12th should be a singly-indirect block \(just like the current one\); the 13th should be your new doubly-indirect block.

## Solution

![](../.gitbook/assets/image%20%289%29.png)

### Preliminaries

`mkfs` initializes the file system to have fewer than 2000 free data blocks, too few to show off the changes you’ll make. Modify `kernel/param.h` to change `FSSIZE` from 2000 to 200,000:

```text
#define FSSIZE       200000  // size of file system in blocks
```

Rebuild `mkfs` so that is produces a bigger disk: `$ rm mkfs/mkfs fs.img; make mkfs/mkfs`

### Change file metadata

Change NDIRECT from 12 to 11. In `file.h`, modify in-memory copy of an `inode` struct: Change size of `addrs` from `NDIRECT+1` to `NDIRECT+2`.

### Add features in `bmap`

`bmap` return the disk block address of the nth block in inode ip. If there is no such block, bmap allocates one.

in `fs.h`

```text
#define NDIRECT 11
#define NINDIRECT (BSIZE / sizeof(uint))

#define NDOUBLY_INDIRECT NINDIRECT*NINDIRECT
#define MAXFILE (NDIRECT + NINDIRECT + NDOUBLY_INDIRECT)
```

Improve `bmap` to support this double indirect: 1. Load double indirect block, allocating if necessary. 2. Load 2nd layer block. 3. Find disk block.

```c
  bn -= NINDIRECT;

  if(bn < NDOUBLY_INDIRECT){
    // Load double indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT + 1]) == 0)
      ip->addrs[NDIRECT + 1] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;

    // load 2nd layer block.
    uint double_index = bn / NINDIRECT;
    if((addr = a[double_index]) == 0){
      a[double_index] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);

    // now find disk block.
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    uint pos = bn % NINDIRECT;
    if ((addr = a[pos]) == 0) {
      a[pos] = addr = balloc(ip->dev);
      log_write(bp);
    }
```

## 心得

画图是最好的解答之一，再把画出的设计图实现，就是编程。

