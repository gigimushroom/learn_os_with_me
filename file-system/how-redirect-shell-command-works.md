# How Redirect Shell command works

## system/kernel/filesystem

`echo hi > result.txt` Write ‘hi’ to result.txt file.

`cat < words.txt` Cat program reads from words.txt file.

That’s the part we all know.

### But, how it works internally in OS?

### Shell Parsing

Parsing logic must identify a redirect command, and treats them differently than normal commands, pipe, etc. **If we have ‘&lt;‘, it is reading the file on right side, feed to the program running on left side.** We create a COMMAND struct with: 1. fd = 0 2. mode is O\_RDONLY. 3. command type is redirect

Same thing **for ‘&gt;’, which is for writing what we output in left, to a file on the right.** We create a COMMAND struct with: 1. fd = 1 2. mode is O\_WRONLY\|O\_CREATE. 3. command type is redirect

### Shell Running Command

```c
case REDIR:
  rc = (struct redircmd *) cmd;
  close(rc->fd);
  if(open(rc->file, rc->mode) < 0){
     fprintf(2, “open %s failed\n”, rc->file);
     exit(1);
  }
  runcmd(rc->cmd);
```

1. Close the corresponding file descriptor.
2. Open file with the set mode, specified file name. The file will use the fd we just closed.
3. Run the Command, which is the one on the left side. Ex: 'echo hi' or 'cat'.

   Note:

   Case fd == 0:

   `Open` cmd opens file with fd 0, which is standard input. Thus, when cat needs to read from standard input, it read from the file linked to fd, which is our file.

   Case fd = = 1:

   `Open` cmd opens file with fd 1, which is standard output. Thus, when `echo hi` wants to write to standard output, it writes to the file \(result.txt\) we specified.

### How on earth is file descriptor? Is it some secrets that OS didn't tell us?

#### Concepts

1. In OS, most resources are represented as files, including deivces, pipes, and real files. This is provided by the file descriptor layer.
2. Each process has its own table of open files, or file descriptors.
3. Each open file is defined below

```c
struct file {
enum { FD_NONE, FD_PIPE, FD_INODE, FD_DEVICE } type;
int ref;*// reference count*
char readable;
char writable;
struct pipe *pipe;*// FD_PIPE*
struct inode *ip;*// FD_INODE and FD_DEVICE*
uint off;*// FD_INODE and FD_DEVICE*
short major;*// FD_DEVICE*
short minor;*// FD_DEVICE*
};
```

#### How System Call Open works?

Each call to `open` creates a new open file. Since each CPU process has its list of open files, it will add the recently open file to its table, and return the index position of the file to user program. That’s why we see a fd integer in our user program after calling `open` Internally, `fd` in OS is linked to a file struct, indicating which file, permission, offset, and in memory node info \(`inode`\), reference count, file type.

## Summary

`echo hi > result.txt` Create a file called ‘result.txt’ with `fd` is 1 \(standard output\). `echo` always writes its content to standard output \(fd 1\), now redirects and writes to the file.

`cat < words.txt` Open a file named ‘words.txt’ with `fd` is 0 \(standard input\). `cat` always read from standard input, now redirects and read from the file.

