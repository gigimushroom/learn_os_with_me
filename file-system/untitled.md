# FS System Calls

### Create a new name for an existing inode

The functions `sys_link` and `sys_unlink` edit directories, creating or removing references to inodes.

Sys\_link creates a new name for an existing inode

### Flow

1. Begins by fetching its arguments, two strings `old` and `new`.
2. Assuming old exists and is not a directory.
3. `sys_link` increments old path’s inode’s `ip->nlink` count. 
4. Then `sys_link` calls `nameiparent` to find the parent directory and final path element of new.
5. Creates a new directory entry pointing at old ’s inode.

```c
// Create the path new as a link to the same inode as old.
uint64
sys_link(void)
{
  char name[DIRSIZ], new[MAXPATH], old[MAXPATH];
  struct inode *dp, *ip;

  if(argstr(0, old, MAXPATH) < 0 || argstr(1, new, MAXPATH) < 0)
    return -1;

  begin_op(ROOTDEV);
  if((ip = namei(old)) == 0){
    end_op(ROOTDEV);
    return -1;
  }

  ilock(ip);
  if(ip->type == T_DIR){
    iunlockput(ip);
    end_op(ROOTDEV);
    return -1;
  }

  ip->nlink++;
  iupdate(ip);
  iunlock(ip);

  if((dp = nameiparent(new, name)) == 0)
    goto bad;
  ilock(dp);
  if(dp->dev != ip->dev || dirlink(dp, name, ip->inum) < 0){
    iunlockput(dp);
    goto bad;
  }
  iunlockput(dp);
  iput(ip);

  end_op(ROOTDEV);

  return 0;

bad:
…
}
```

### Create a new name for a new inode

The function `create` creates a new name for a new inode.

`open` is used in `sys_open`, `sys_mkdir`, and `sys_mknod`.

#### Flow

1. Find parent inode from path.
2. Lock parent inode and fetch content from disk.
3. Call directory lookup to check if the about-to-create inode already exists.
4. If yes, returns.
5. Otherwise, continue to allocate a new on-disk inode. 
6. Allocation of on-disk inode returns a copy of inode in cache.
7. Update inode metadata.
8. Save back to disk using txn.
9. If the node type is directory, add links for `.`, and `..`.
10. Otherwise, install link in the parent inode, with name, and inode index.

```c
static struct inode*
create(char *path, short type, short major, short minor)
{
  struct inode *ip, *dp;
  char name[DIRSIZ];

  if((dp = nameiparent(path, name)) == 0)
    return 0;

  ilock(dp);

  if((ip = dirlookup(dp, name, 0)) != 0){
    iunlockput(dp);
    ilock(ip);
    if(type == T_FILE && (ip->type == T_FILE || ip->type == T_DEVICE))
      return ip;
    iunlockput(ip);
    return 0;
  }

  if((ip = ialloc(dp->dev, type)) == 0)
    panic(“create: ialloc”);

  ilock(ip);
  ip->major = major;
  ip->minor = minor;
  ip->nlink = 1;
  iupdate(ip);

  if(type == T_DIR){  // Create . and .. entries.
    dp->nlink++;  // for “..”
    iupdate(dp);
    // No ip->nlink++ for “.”: avoid cyclic ref count.
    if(dirlink(ip, “.”, ip->inum) < 0 || dirlink(ip, “..”, dp->inum) < 0)
      panic(“create dots”);
  }

  if(dirlink(dp, name, ip->inum) < 0)
    panic(“create: dirlink”);

  iunlockput(dp);

  return ip;
}
```

### The famous `open` system call

1. If CREATE mode, call above `create` to get a new locked inode.
2. Else, call `namei` to get an unlocked inode from the path. Lock it after.
3. Allocate file and fd, update metadata. Includes set the inode, and other type, offset, etc.
4. Unlock the inode at the end.

```c
uint64
sys_open(void)
{
  char path[MAXPATH];
  int fd, omode;
  struct file *f;
  struct inode *ip;
  int n;

  if((n = argstr(0, path, MAXPATH)) < 0 || argint(1, &omode) < 0)
    return -1;

  begin_op(ROOTDEV);

  if(omode & O_CREATE){
    ip = create(path, T_FILE, 0, 0);
    if(ip == 0){
      end_op(ROOTDEV);
      return -1;
    }
  } else {
    if((ip = namei(path)) == 0){
      end_op(ROOTDEV);
      return -1;
    }
    ilock(ip);
    if(ip->type == T_DIR && omode != O_RDONLY){
      iunlockput(ip);
      end_op(ROOTDEV);
      return -1;
    }
    printf(“Open ….Path(%s)\n”, path);
  }

  if(ip->type == T_DEVICE && (ip->major < 0 || ip->major >= NDEV)){
    iunlockput(ip);
    end_op(ROOTDEV);
    return -1;
  }

  if((f = filealloc()) == 0 || (fd = fdalloc(f)) < 0){
    if(f)
      fileclose(f);
    iunlockput(ip);
    end_op(ROOTDEV);
    return -1;
  }

  if(ip->type == T_DEVICE){
    printf(“file typ is device. Path(%s)\n”, path);
    f->type = FD_DEVICE;
    f->major = ip->major;
    f->minor = ip->minor;
  } else {
    f->type = FD_INODE;
  }
  f->ip = ip;
  f->off = 0;
  f->readable = !(omode & O_WRONLY);
  f->writable = (omode & O_WRONLY) || (omode & O_RDWR);

  iunlock(ip);
  end_op(ROOTDEV);

  return fd;
}
```

