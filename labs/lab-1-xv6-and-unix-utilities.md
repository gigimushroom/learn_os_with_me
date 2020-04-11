---
description: Implement some user tools.
---

# Lab 1 Xv6 and Unix utilities

## Introduction 

This lab makes you familiar with xv6 and its system calls.

## Sleep

Implement the UNIX program sleep for xv6; your sleep should pause for a user-specified number of ticks. \(A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip.\) Your solution should be in the file user/sleep.c.

### Solution

```text
if(argc < 2){
  fprintf(2, “Usage: sleep for a new sec…\n”);
  exit(-1);
}
sleep(atoi(argv[1]));
exit(1);
```

## pingpong

Write a program that uses UNIX system calls to \`\`ping-pong’’ a byte between two processes over a pair of pipes, one for each direction. The parent sends by writing a byte to parent\_fd\[1\] and the child receives it by reading from parent\_fd\[0\]. After receiving a byte from parent, the child responds with its own byte by writing to child\_fd\[1\], which the parent then reads. Your solution should be in the file user/pingpong.c.

### Solution

```text
int p[2];
  char buf[100];
  pipe(p);

  int pid = fork();
  if (pid == 0) {
    write(p[1], “ping”, 4);
    printf(“Thread id %d: received ping\n”, getpid());
  } 
  else {
    wait(0);
    read(p[0], buf, 4);
    printf(“Thread id %d: received pong\n”, getpid());
  }
```

## primes

Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file user/primes.c.

### Hints

```text
$ primes
prime 2
prime 3
prime 5
prime 7
prime 11
prime 13
prime 17
prime 19
prime 23
prime 29
prime 31
```

### Solution

Use pipe and fork to set up the pipeline. The first process feeds the numbers 2 through 35 into the pipeline. For each prime number, you will arrange to create one process that reads from its left neighbour over a pipe and writes to its right neighbour over another pipe.

![](../.gitbook/assets/image%20%2843%29.png)

Loop from 2 to 35. Write them to pipe one by one. Read the first one from pipe buffer. Record as `Num`. Create a new pipe, write everything not dividable by `Num` \(Use mod\).

```c
void exec_pipe(int *fd*)
{
    int num;
    read(fd, &num, 4);
    printf(“Thead(%d) prime %d\n”, getpid(), num);

    int p[2];
    pipe(p);
    int tmp = -1;
    while (1) {
        int n = read(fd, &tmp, 4);
        if (n<= 0) {
            break;
        }
        if (tmp % num != 0) {
            *//printf(“%d writing %d and n is: %d\n”, getpid(), tmp, n);*
            write(p[1], &tmp, 4);
        }
    }
    if (tmp == -1) {
        close(p[1]);
        close(p[0]);
        close(fd);
        return;
    }
    int pid = fork();
    if (pid == 0) {
        close(p[1]);
        close(fd);
        exec_pipe(p[0]);
        close(p[0]);
    }
    else {
        close(p[1]);
        close(p[0]);
        close(fd);
        wait(0);
    }
}
```

```c
int
main(int *argc*, char **argv*[])
{
  int p[2];
  pipe(p);
  for (int i = 2; i<35; i++) {
      int n = i;
      write(p[1], &n, 4);
  }
  close(p[1]);
  exec_pipe(p[0]);
  close(p[0]);

  exit(1);
}
```

## find

Write a simple version of the UNIX find program: find all the files in a directory tree whose name matches a string. Your solution should be in the file user/find.c.

### Solution

Code is modified based on `ls`. Below is the `ls` code:

```c
char*
fmtname(char **path*)
{
  static char buf[DIRSIZ+1];
  char *p;

  *// Find first character after last slash.*
  for(p=path+strlen(path); p >= path && *p != ‘/‘; p—)
    ;
  p++;

  *// Return blank-padded name.*
  if(strlen(p) >= DIRSIZ)
    return p;
  memmove(buf, p, strlen(p));
  memset(buf+strlen(p), ‘ ‘, DIRSIZ-strlen(p));
  return buf;
}

void
ls(char **path*)
{
  char buf[512], *p;
  int fd;
  struct dirent de;
  struct stat st;

  if((fd = open(path, 0)) < 0){
    fprintf(2, “ls: cannot open %s\n", path);
    return;
  }

  if(fstat(fd, &st) < 0){
    fprintf(2, “ls: cannot stat %s\n”, path);
    close(fd);
    return;
  }

  switch(st.type){
  case T_FILE:
    printf(“%s %d %d %l\n”, fmtname(path), st.type, st.ino, st.size);
    break;

  case T_DIR:
    if(strlen(path) + 1 + DIRSIZ + 1 > sizeof buf){
      printf(“ls: path too long\n”);
      break;
    }
    strcpy(buf, path);
    p = buf+strlen(buf);
    *p++ = '/';
    while(read(fd, &*de*, sizeof(de)) == sizeof(de)){
      if(de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);
      p[DIRSIZ] = 0;
      if(stat(buf, &st) < 0){
        printf(“ls: cannot stat %s\n”, buf);
        continue;
      }
      printf(“%s %d %d %d\n”, fmtname(buf), st.type, st.ino, st.size);
    }
    break;
  }
  close(fd);
}
```

Modify the case `T_DIR` to do the filename comparision.

```c
if (st.type == T_DIR && *p != '.') {
          ls(p, filename);
      } else if (strcmp(p, filename) == 0) {
        printf(“./%s\n”, buf);
      }
```

## xargs

Write a simple version of the UNIX xargs program: read lines from standard input and run a command for each line, supplying the line as arguments to the command. Your solution should be in the file user/xargs.c.

### Hints

* Use fork and exec system call to invoke the command on each line of input. Use wait in the parent to wait for the child to complete running the command.
* Read from stdin a character at the time until the newline character \(‘\n’\).
* kernel/param.h declares `MAXARG`, which may be useful if you need to declare an argv.
* Changes to the file system persist across runs of qemu; to get a clean file system run make clean and then make qemu.

### Solution

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/param.h"

int
getcmd(char **buf*, int *nbuf*)
{
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) *// EOF*
  {
    *//printf("WOW!\n");*
    return -1;
  }

  return 0;
}

char whitespace[] = " \t\r\n\v";

int
gettoken(char ***ps*, char **es*, char ***q*, char ***eq*)
{
  char *s;
  int ret;

  s = *ps;
  while(s < es && strchr(whitespace, *s))
    s++;
  if(q)
    *q = s;
  ret = *s;
  switch(*s){
  case 0:
    break;
  default:
    ret = 'a';
    while(s < es && !strchr(whitespace, **s*))
      s++;
    break;
  }
  if(eq)
    *eq = s;

  while(s < es && strchr(whitespace, *s))
    s++;
  *ps = s;
  return ret;
}

int
main(int *argc*, char **argv*[])
{
  char *xargs[MAXARG];
  for (int i=1; i<argc;i++) {
    *// Skip `xargs` cmd name.*
    xargs[i-1] = argv[i];
  }

  static char buf[MAXARG][100];
  char *q, *eq;
  int j = argc-1;
  int i = 0;
  *// Split each line into array of args.*
  while(getcmd(buf[i], sizeof(buf[i])) >= 0){
    char *s = buf[i];
    char *es = s + strlen(s);
    while (gettoken(&s, es, &q, &eq) != 0) {
      *// Set end to 0.*
      xargs[j] = q;
      *eq = 0;
      j++;
      i++;
    }
  }

  int pid = fork();
  if (pid == 0) {
    exec(xargs[0], xargs);
  }
  wait();

  exit();
}
```

### Things learnt

1. `ls` looks up current process directory if no args is passed in. Similar to `ls .` `namex` is used to look up directory `inode`, and find all filenames in the directory `inode`.
2. `pipe` does not close if write part fd is closed. So we can do: write everything to `p[1]`, close `p[1]’s fd`, then read from `p[0]`.
3. Think more why we designed this way, not just make it working and leave.

## 心得

C程序短小精悍。在unix的世界里数个小部件拼接成大项目。每一个bit都来的有缘由, 没有多余的内存用来浪费。这才是编程的精髓吧。

