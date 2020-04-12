# Pipes

## Overview

Pipes provide a way for processes to communicate. An example running `wc` with standard input connected to the read end of pipe:

```c
int p[2];
char *argv[2];
argv[0] = “wc”;
argv[1] = 0;
pipe(p);
if(fork() == 0) {
  close(0);
  dup(p[0]);
  close(p[0]);
  close(p[1]);
  exec(“/bin/wc”, argv);
} else {
  close(p[0]);
  write(p[1], “hello world\en”, 12);
  close(p[1]);
}
```

`dup(p[0])` : The child dups the read end onto file descriptor 0. When wc reads from its standard input, it reads from the pipe.

## Diagram

![](../.gitbook/assets/image%20%281%29.png)

close\(0\) closes the fd 0. Make it available for next request. `dup` makes fd 0 pointing pipe read side. `fd` is just an index within process, points to a file opened by process. `fd` could be changed to point to different file. Any program wants to read from fd 0 \(standard input\) now read from pipe read side.

## Pipe Read, Write, Close

#### Read side of pipe

If no data is available, a read on a pipe waits for either data to be written or all file descriptors referring to the write end to be closed.

#### Pipe close

When all file descriptors for write side of pipe closed, the pipe write side is considered to be closed. It wakens up read side to perform one more read. If buffer contains nothing, `piperead` returns 0 just like EOF.

```c
void
pipeclose(struct pipe *pi, int writable)
{
  acquire(&pi->lock);
  if(writable){
    pi->writeopen = 0;
    wakeup(&pi->nread);  // HERE!
  } else {
    pi->readopen = 0;
    wakeup(&pi->nwrite);
  }
  if(pi->readopen == 0 && pi->writeopen == 0){
    release(&pi->lock);
    kfree((char*)pi);
  } else
    release(&pi->lock);
}
```

{% page-ref page="../scheduling/how-unix-pipes-work.md" %}

## How shell implements pipelines

```c
  case PIPE:
    pcmd = (struct pipecmd*)cmd;
    if(pipe(p) < 0)
      panic(“pipe”);
    if(fork1() == 0){
      close(1);
      dup(p[1]);
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->left);
    }
    if(fork1() == 0){
      close(0);
      dup(p[0]);
      close(p[0]);
      close(p[1]);
      runcmd(pcmd->right);
    }
    close(p[0]);
    close(p[1]);
    wait(0);
    wait(0);
    break;
```

`echo hi | wc` left side writes to `fd` `1`. Right side reads from `fd 0`.

