---
description: Implement a primitive form of user-level interrupt/fault handlers.
---

# Lab 6 Alarm

## Introduction 

In this exercise you'll add a feature to xv6 that periodically alerts a process as it uses CPU time. This might be useful for compute-bound processes that want to limit how much CPU time they chew up, or for processes that want to compute but also want to take some periodic action. More generally, you’ll be implementing a primitive form of user-level interrupt/fault handlers; you could use something similar to handle page faults in the application, for example.

## Task

Add a new `sigalarm(interval, handler)` system call. If an application calls `sigalarm(n, fn)`, then after every n “ticks” of CPU time that the program consumes, the kernel should cause application function `fn` to be called. When `fn` returns, the application should resume where it left off. A tick is a fairly arbitrary unit of time in xv6, determined by how often a hardware timer generates interrupts.

 [Lab: user-level threads and alarm](https://pdos.csail.mit.edu/6.828/2019/labs/syscall.html)

### Hints

{% hint style="info" %}
Test code Call `sigalarm` to set up alarm so that call function `periodic` after 2 ticks. `periodic` is the handler code after alarm is fired. It calls `periodic` at the end to reset process state to before the alarm fired.
{% endhint %}

{% code title="Test code" %}
```c
void
periodic()
{
  count = count + 1;
  printf("alarm!\n");
  periodic();
}

// tests whether the kernel calls
// the alarm handler even a single time.
void
test0()
{
  int i;
  printf("test0 start\n");
  count = 0;
  sigalarm(2, periodic);
  for(i = 0; i < 1000*500000; i++){
    if((i % 1000000) == 0)
      write(2, ".", 1);
    if(count > 0)
      break;
  }
  sigalarm(0, 0);
  if(count > 0){
    printf("test0 passed\n");
  } else {
    printf("\ntest0 failed: the kernel never called the alarm handler\n");
  }
}
```
{% endcode %}

## Solution

### Add 2 system calls

```c
int sigalarm(int ticks, void (*handler)());
int sigreturn(void);
```

### Add alarm infos in Process state

```c
  uint64 handler;
  int ticks;
  int cur_ticks;
  struct trapframe *alarm_tf; // cache the trapframe when timer fires
  int alarm_on;
```

Initialize cur\_ticks in `allocproc` \(process allocation method\). `p->cur_ticks = 0;`

### Handler alarm in trap handler

Save trapframe snapshot in process new field `alarm_tf` Increment ticks. If ticks is reaching max, set `epc` program counter to the handler, so handler is called once trap is returned to user space.

```c
if((which_dev = devintr()) != 0){
    // lab 6 alarm
    if (which_dev == 2 && p->alarm_on == 0) {
      // Save trapframe
      p->alarm_on = 1;
      struct trapframe *tf = kalloc();
      memmove(tf, p->tf, PGSIZE);
      p->alarm_tf = tf;

      p->cur_ticks++;
        if (p->cur_ticks >= p->ticks)
        p->tf->epc = p->handler;
    }
```

### Implement system calls

`sigalarm` saves function handler and max ticks to process state.

```c
uint64 sys_sigalarm(void)
{
  uint64 addr;
  int ticks;

  if(argint(0, &ticks) < 0)
    return -1;
  if(argaddr(1, &addr) < 0)
    return -1;

  myproc()->ticks = ticks;
  myproc()->handler = addr;

  return 0;
}
```

`sigreturn` is called after alarm is fired, alarm handler is called, and handled by user space. It restores trapframe page from `alarm_tf`. Clean up ticks info.

```c
uint64 sys_sigreturn(void)
{
  struct proc *p = myproc();
  memmove(p->tf, p->alarm_tf, PGSIZE);

  kfree(p->alarm_tf);
  p->alarm_tf = 0;
  p->alarm_on = 0;
  p->cur_ticks = 0;
  return 0;
}
```

