# I/O and File descriptors

A fd is a small integer representing a kernel-managed object that a process may read/write. A process obtains a fd by opening a file, directory, device, pipe, or socket. In unix, a file has different types that what it currently represents.

Every process has a private space of file descriptors starting at zero. Xv6 kernel uses fd as an index into a per-process table.

By convention, fd 1\(standard input\), fd 2\(standard output\), fd3 \(standard error\). `xv6 shell` ensure the 3 fds are open when starting up.

`read` and `write` system calls advance the file by offset \(how much it reads or writes\), so next time, the read/write starts from the new offset.

One interesting aspect is program like `cat` does not know whether it is reading from file, console, socket, or pipe. It does not know whether it is printing to a console, or a file. The use of fds and the conventions allows some simple implementation of user programs.

`close` releases fd, making it free for reuse. **A newly allocated fd is always the lowest-numbered unused descriptor of the current process.** Unix shell utilizes this feature with `fork`, to implement smart I/O redirection, and implement `pipe`.

### I/O redirection

`fork` copies parent’s file descriptor tables, with its memory. The system call `exec` replaces the calling process’ memory but **preserves its file table.**

**Example how command cat &lt; input.txt works**

```c
char *argv[2];
argv[0] = “cat”;
argv[1] = 0;
if(fork() == 0) {
  close(0);
  open(“input.txt”, O_RDONLY);
  exec(“cat”, argv);
}
```

After the child closes file descriptor 0, open is guaranteed to use that file descriptor for the newly opened input.txt: 0 will be the smallest available file descriptor. Cat then executes with file descriptor 0 \(standard input\) referring to input.txt

### `fork` copies fd table, but file offset is shared.

The underlying file offset is shared between parent and child.

```c
if(fork() == 0) {
  write(1, “hello “, 6);
  exit(0);
} else {
  wait(0);
  write(1, “world\en”, 6);
}
```

`dup` system call is similar. Duplicates an existing file descriptor, returning a new fd that refers to the same I/O object. Both file descriptors share the an offset.

```c
fd = dup(1);
write(1, “hello “, 6);
write(fd, “world\en”, 6);
```

#### Do you know why the file offset is shared?

`fd` is basically an index pointing to an actual file object. The file is shared between parent, and child. After `fork` or `dup`, the reference count of the file is incremented.

#### Fun Fun Command

`ls existing-file non-existing-file > tmp1 2>&1` The `2>&1` tells the shell to give the command a file descriptor 2 that is a duplicate of descriptor 1. Both the name of the existing file and the error message for the non-existing file will show up in the file tmp1.

### Summary

File descriptors are a powerful abstraction, because they hide the details of what they are connected to: a process writing to file descriptor 1 may be writing to a file, to a device like the console, or to a pipe.

