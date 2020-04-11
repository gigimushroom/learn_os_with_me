# File Descriptor Layer

### Concept

#### Everything is a file

A cool aspect of the Unix interface is that most resources in Unix are represented as files, including devices such as the console, pipes, and of course, real files,

#### Each process tracks its open file

Xv6 gives each process its own table of open files, or file descriptors.

#### Global table

All the open files in the system are kept in a global file table. The file table can allocate new file, create a duplicate reference, release a reference, and read and write data.

#### Concurrent Access

If multiple processes open the same file independently, the different instances will have different I/O offsets \(Use `dup`\)

### File Data Structure

```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE, FD_SOCK } type;
  int ref; // reference count
  char readable;
  char writable;
  struct net *net;   // FD_NET
  struct pipe *pipe; // FD_PIPE
  struct inode *ip;  // FD_INODE and FD_DEVICE
  struct sock *sock; // FD_SOCK
  uint off;          // FD_INODE and FD_DEVICE
  short major;       // FD_DEVICE
  short minor;       // FD_DEVICE
};
```

### Read and Write

Reading a file begins with checking permission. Ex: pipe has 2 files, one for read, one for write. Based on the file type, it reads differently.

```c
// Read from file f.
// addr is a user virtual address.
int
fileread(struct file *f, uint64 addr, int n)
{
  int r = 0;

  if(f->readable == 0)
    return -1;

  if(f->type == FD_PIPE){
    r = piperead(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(f, 1, addr, n);
  } else if(f->type == FD_INODE){
    ilock(f->ip);
    if((r = readi(f->ip, 1, addr, f->off, n)) > 0)
      f->off += r;
    iunlock(f->ip);
  } else if (f->type == FD_SOCK) {
    r = sockread(f, addr, n);
  } else {
    panic(“fileread”);
  }

  return r;
}
```

```c
// Write to file f.
// addr is a user virtual address.
int
filewrite(struct file *f, uint64 addr, int n)
{
  int r, ret = 0;

  if(f->writable == 0)
    return -1;

  if(f->type == FD_PIPE){
    ret = pipewrite(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].write)
      return -1;
    ret = devsw[f->major].write(f, 1, addr, n);
  } else if(f->type == FD_INODE){
    // write a few blocks at a time to avoid exceeding
    // the maximum log transaction size, including
    // i-node, indirect block, allocation blocks,
    // and 2 blocks of slop for non-aligned writes.
    // this really belongs lower down, since writei()
    // might be writing a device like the console.
    int max = ((MAXOPBLOCKS-1-1-2) / 2) * BSIZE;
    int I = 0;
    while(I < n){
      int n1 = n - I;
      if(n1 > max)
        n1 = max;

      begin_op(f->ip->dev);
      ilock(f->ip);
      if ((r = writei(f->ip, 1, addr + I, f->off, n1)) > 0)
        f->off += r;
      iunlock(f->ip);
      end_op(f->ip->dev);

      if(r < 0)
        break;
      if(r != n1)
        panic(“short filewrite”);
      I += r;
    }
    ret = (I == n ? n : -1);
  } else if (f->type == FD_SOCK) {
    ret = sockwrite(f, addr, n);
  } else {
    panic(“filewrite”);
  }

  return ret;
}
```

