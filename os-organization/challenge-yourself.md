# Challenge yourself

## Problem

In `proc.c`, there is an array `initcode[]` of binary code. What is its relationship with `initcode.S`?

## Solution

### Research

Testing code: `-xc` to make the hex dump more readable. Example:

```text
$ od -xc test.txt
0000000      6568    6c6c    0a6f
           h   e   l   l   o  \n
0000006
```

The `od` Command

![](../.gitbook/assets/image%20%2839%29.png)

`od` man page

![](../.gitbook/assets/image%20%2841%29.png)

`-t` is the output format. `x` is hex. `c` is chars in default char set.

Note: Each **Hexadecimal** character represents 4 bits \(0 - 15 decimal\). A byte is 2 hex.

### Answer

We want a memory dump of instructions in hex format, and separated by each char. 

We have to load `initcode`, and use it to call system call `exec` to run `init`. We cannot directly load `init` binary as hex dump, otherwise we have to do something similar to what `exec` does, set up C stacks, parse ELF headers, etc. 

So the simply solution is: 

1. use `objcopy` to copy a stripped instruction only data file. 

2. Use `od` to print the data file in hex, separated by byte \(char\). 

3. Append `0x` to each char. 

4. That is the result if `initcode` array you see in `proc.c`.



**References**

`od` - dump files in various formats. 

See [od](https://pubs.opengroup.org/onlinepubs/9699919799/utilities/od.html) [The UNIX School: 3 different ways of dumping hex contents of a file](http://www.theunixschool.com/2011/06/3-different-ways-of-dumping-hex.html)

## Challenge!

Try to switch the hex dump of `initcode` to a program you wrote!

I modify `initcode.S` to use `echo`:

```c
#include “syscall.h”

# exec(init, argv)
.globl start
start:
        la a0, echo
        la a1, argv
        li a7, SYS_exec
        ecall

# for(;;) exit();
exit:
        li a7, SYS_exit
        ecall
        jal exit

# char init[] = “/init\0”;
echo:
  .string “/echo\0”

# char *argv[] = { init, 0 };
.p2align 2
argv:
  .long echo
  .long 0
```

After running `make`, get the hex dump of the binary:

```text
$ od -t xC initcode
0000000    17  05  00  00  13  05  05  02  97  05  00  00  93  85  05  02
0000020    9d  48  73  00  00  00  89  48  73  00  00  00  ef  f0  bf  ff
0000040    2f  65  63  68  6f  00  00  01  20  00  00  00  00  00  00  00
0000060    00  00  00
0000063
```

Change the array in `proc.c`:

```c
// a user program that calls exec(“/echo”)
// od -t xC initcode
uchar initcode[] = {
  0x17, 0x05, 0x00, 0x00, 0x13, 0x05, 0x05, 0x02,
  0x97, 0x05, 0x00, 0x00, 0x93, 0x85, 0x05, 0x02,
  0x9d, 0x48, 0x73, 0x00, 0x00, 0x00, 0x89, 0x48,
  0x73, 0x00, 0x00, 0x00, 0xef, 0xf0, 0xbf, 0xff,
  0x2f, 0x65, 0x63, 0x68, 0x6f, 0x00, 0x00, 0x01,
  0x20, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
  0x00, 0x00, 0x00
};
```

Make a special `echo`:

```c
#include “kernel/types.h"
#include “kernel/stat.h”
#include “user/user.h”
#include “kernel/fcntl.h”

int
main(void)
{

  if(open(“console”, O_RDWR) < 0){
    mknod(“console”, 1, 1);
    open(“console”, O_RDWR);
  }
  dup(0);  // stdout
  dup(0);  // stderr

  for(;;){
    printf(“Mushroom os is the best!\n”);
  }
}
```

As a result, your terminal prints the following forever:

```text
Mushroom os is the best!
Mushroom os is the best!
Mushroom os is the best!
Mushroom os is the best!
Mushroom os is the best!
Mushroom os is the best!
…
```

**We hacked the kernel to run our program in the very first user process!**

