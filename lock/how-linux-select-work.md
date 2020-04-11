# How linux select work

[select\(2\) - Linux manual page](http://man7.org/linux/man-pages/man2/select.2.html) 

select, pselect, FD\_CLR, FD\_ISSET, FD\_SET, FD\_ZERO - synchronous I/O multiplexing

```c
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);

void FD_CLR(int fd, fd_set *set);
int  FD_ISSET(int fd, fd_set *set);
void FD_SET(int fd, fd_set *set);
void FD_ZERO(fd_set *set);
```

`select()` and `pselect()` allow a program to monitor multiple file descriptors, waiting until one or more of the file descriptors become “ready” for some class of I/O operation \(e.g., input possible\).

The `timeout` argument specifies the interval that select\(\) should block waiting for a file descriptor to become ready. The call will block until either:

* a file descriptor becomes ready;
* the call is interrupted by a signal handler; or
* the timeout expires.

### Example

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>
#include <sys/types.h>
#include <unistd.h>

int
main(void)
{
   fd_set rfds;
   struct timeval tv;
   int retval;

   /* Watch stdin (fd 0) to see when it has input. */
   FD_ZERO(&rfds);
   FD_SET(0, &rfds);

   /* Wait up to five seconds. */
   tv.tv_sec = 5;
   tv.tv_usec = 0;

   retval = select(1, &rfds, NULL, NULL, &tv);
   /* Don't rely on the value of tv now! */

   if (retval == -1)
       perror(“select()”);
   else if (retval)
       printf(“Data is available now.\n”);
       /* FD_ISSET(0, &rfds) will be true. */
   else
       printf(“No data within five seconds.\n”);

   exit(EXIT_SUCCESS);
}
```

### Linux Implementation

Code is from linux 2.6.

This is the `select` system call entry.

```c
SYSCALL_DEFINE5(select, int, n, fd_set __user *, inp, fd_set __user *, outp,
        fd_set __user *, exp, struct timeval __user *, tvp)
```

The core logic

```c
int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
               fd_set __user *exp, struct timespec *end_time)
```

Allocate bit maps, all fds from 3 registered fd sets have a bit presented.

```c
    /*
     * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
     * since we used fdset we need to allocate memory in units of
     * long-words. 
     */
    size = FDS_BYTES(n);
    bits = stack_fds;
    if (size > sizeof(stack_fds) / 6) {
        /* Not enough space in on-stack array; must use kmalloc */
        ret = -ENOMEM;
        bits = kmalloc(6 * size, GFP_KERNEL);
        if (!bits)
            goto out_nofds;
    }
    fds.in      = bits;
    fds.out     = bits +   size;
    fds.ex      = bits + 2*size;
    fds.res_in  = bits + 3*size;
    fds.res_out = bits + 4*size;
    fds.res_ex  = bits + 5*size;

    if ((ret = get_fd_set(n, inp, fds.in)) ||
        (ret = get_fd_set(n, outp, fds.out)) ||
        (ret = get_fd_set(n, exp, fds.ex)))
        goto out;
    zero_fd_set(n, fds.res_in);
    zero_fd_set(n, fds.res_out);
    zero_fd_set(n, fds.res_ex);

    ret = do_select(n, &fds, end_time);
```

Loop and fetch each file, check its state, mark the bit if qualified. `int do_select(int n, fd_set_bits *fds, struct timespec *end_time)`

```java
file = fget_light(i, &fput_needed);
if (file) {
    f_op = file->f_op;
    mask = DEFAULT_POLLMASK;
    if (f_op && f_op->poll) {
        wait_key_set(wait, in, out, bit);
        mask = (*f_op->poll)(file, wait);
    }
    fput_light(file, fput_needed);
    if ((mask & POLLIN_SET) && (in & bit)) {
        res_in |= bit;
        retval++;
        wait = NULL;
    }
    if ((mask & POLLOUT_SET) && (out & bit)) {
        res_out |= bit;
        retval++;
        wait = NULL;
    }
    if ((mask & POLLEX_SET) && (ex & bit)) {
        res_ex |= bit;
        retval++;
        wait = NULL;
    }
}
```

Each file will be set state accordingly after I/O. The states are defined as:

```c
#define POLLIN_SET (POLLRDNORM | POLLRDBAND | POLLIN | POLLHUP | POLLERR)
#define POLLOUT_SET (POLLWRBAND | POLLWRNORM | POLLOUT | POLLERR)
#define POLLEX_SET (POLLPRI)
```

### Summary

`select` code checks every single file, and return results. We have to use `FD_ISSET(FD, &rfds)` to check which fd is actually ready. 

If a lot of fds we are monitoring, the cost is very heavy.

