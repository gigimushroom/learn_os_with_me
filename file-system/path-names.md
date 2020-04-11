# Path names

Path name lookup involves a succession of calls to `dirlookup`, one for each path component.

### Look up and return the inode for a path name

After called `skipelem`, at the beginning of each iteration,: `inode` points to the parent node.  
`path` points to next lookup path. `name` points to the current directory name. If we want to return parent, we just return the current `inode`. Otherwise, we find the inode of ‘name\`, and return it.

```c
// Look up and return the inode for a path name.
// If parent != 0, return the inode for the parent and copy the final
// path element into name, which must have room for DIRSIZ bytes.
// Must be called inside a transaction since it calls iput().
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

Helper

```c
// Copy the next path element from path into name.
// Return a pointer to the element following the copied one.
// The returned path has no leading slashes,
// so the caller can check *path==‘\0’ to see if the name is the last one.
// If no name to remove, return 0.
//
// Examples:
//   skipelem(“a/bb/c”, name) = “bb/c”, setting name = “a”
//   skipelem(“///a//bb”, name) = “bb”, setting name = “a”
//   skipelem(“a”, name) = “”, setting name = “a”
//   skipelem(“”, name) = skipelem(“////“, name) = 0
//
static char*
skipelem(char *path, char *name)
{
  char *s;
  int len;

  while(*path == ‘/‘)
    path++;
  if(*path == 0)
    return 0;
  s = path;
  while(*path != ‘/‘ && *path != 0)
    path++;
  len = path - s;
  if(len >= DIRSIZ)
    memmove(name, s, DIRSIZ);
  else {
    memmove(name, s, len);
    name[len] = 0;
  }
  while(*path == ‘/‘)
    path++;
  return path;
}
```

### My Drawing

Find path “a/b/c". 3 iterations. 

![Example of finding path](../.gitbook/assets/image%20%2827%29.png)

### Races and deadlock

Only lock one inode at a time, for getting the next directory, only reference count is incremented.

#### Avoiding deadlock.

For example, `next` points to the same inode as `ip` when looking up “.”. Locking `next` before releasing the lock on `ip` would result in a deadlock. To avoid this deadlock, `namex` unlocks the directory before obtaining a lock on next. Here again we see why the separation between `iget` and `ilock` is important.

