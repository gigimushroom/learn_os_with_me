---
description: Add symbolic links to xv6
---

# Lab 8 File System: Symbolic links

## Introduction 

In this exercise you will add symbolic links to xv6. Symbolic links \(or soft links\) refer to a linked file by pathname; when a symbolic link is opened, the kernel follows the link to the referred file. Symbolic links resembles hard links, but hard links are restricted to pointing to file on the same disk, while symbolic links can cross disk devices. Although xv6 doesn’t support multiple devices, implementing this system call is a good exercise to understand how pathname lookup works.

## Task

Implement the symlink\(char _target, char_ path\) system call, which creates a new symbolic link at linkpath that refers to file named by target.

## Solution

### Create system call, add new file type and new flag

Create `symlink` system call. Add a new file type \(T\_SYMLINK\) to kernel/stat.h to represent a symbolic link. Add a new flag to kernel/fcntl.h, \(O\_NOFOLLOW\), that can be used with the open system call.

### Implement the `symlink(target, path)` system call

I decided to store the target path of a symbolic link in the `inode`’s data blocks. Create a inode of T\_SYMLINK type.

```c
uint64
sys_symlink(void)
{
  char target[MAXPATH], path[MAXARG];
  if(argstr(0, target, MAXPATH) < 0 || argstr(1, path, MAXPATH) < 0){
    return -1;
  }
  //printf(“creating a sym link. Target(%s). Path(%s)\n”, target, path);

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

```c
static struct inode*
create(char *path, short type, short major, short minor)
{
  …
  if((ip = dirlookup(dp, name, 0)) != 0){
     …
    if(type == T_SYMLINK) {
      return ip;
    }
```

### Modify `open` system call to work with symbolic link

1. If the file does not exist, open must fail. 
2. When a process specifies O\_NOFOLLOW in the flags to open, open should open the symlink \(and not follow the symbolic link\).
3. Otherwise, open follows the symlink.
4. If the linked file is also a symbolic link, you must recursively follow it until a non-link file is reached. If the links form a cycle, you must return an error code.

   **`sys_open`**

   ```c
   if ((ip->type == T_SYMLINK) && !(omode & O_NOFOLLOW)){
    int count = 0;
    while (ip->type == T_SYMLINK && count < 10) {
      int len = 0;
      readi(ip, 0, (uint64)&len, 0, sizeof(int));

      if(len > MAXPATH)
        panic(“open: corrupted symlink inode”);

      readi(ip, 0, (uint64)path, sizeof(int), len + 1);
      iunlockput(ip);
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

   `namei` : Look up and return the inode for a path name.

### Thoughts

The key is _where we store the target path._ We should store it in `inode`’s data blocks. By using `readi` and `writei` , we could read and write into/from the `inode->data[0]’s block`

_Bad decision I made_: I originally tried to add a new char array in `inode` in memory file struct, but it does not use buffer cache, not sync to `dinode`, not logged properly, and not store back to disk properly.

### Reference: How `readi` works

A bit more on `readi` and `writei`

```c
// Read data from inode.
// Caller must hold ip->lock.
// If user_dst==1, then dst is a user virtual address;
// otherwise, dst is a kernel address.
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

`bmap(ip, off/BSIZE)` returns the disk block address of the nth block in inode ip. `readi` starts from offset, get the buffer containing the data, copy data from buffer’s data to the destination address. `writei` works similar. Write data from src to inode

#### How `bmap` works

```c
// Inode content
// The content (data) associated with each inode is stored
// in blocks on the disk. The first NDIRECT block numbers
// are listed in ip->addrs[].  The next NINDIRECT blocks are
// listed in block ip->addrs[NDIRECT].

// Return the disk block address of the nth block in inode ip.
// If there is no such block, bmap allocates one.
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

#### How bread works

```c
// Return a locked buf with the contents of the indicated block.
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

## 心得

软连接和硬链接的不同是什么呢？

Term `inode` in OS represents an actual file in disk. A directory contains multiple directory files, each has an `inode` number, and a file name. The number is its actual file's `inode` id.

_A hard link is an additional name for an existing file_. It is a directory file which points to the actual file, and has its own name. The `inode` in OS has metadata to track the number of links to this file. 

File\(`inode`\) can be freed if both number of hard links and number of references are 0.

软连接就是一个可有可无的索引，虽然也是一个存在directory下面的directory file，但是所指的目标可以不存在，或者在另一个device.

硬链接就是真正的恋人关系，是真的存在，生活中都有彼此的印记。

软连接是暗恋，花开无泪，暗恋无声。说的就是14岁的你，是吗？

