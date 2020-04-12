---
description: Design the context switch mechanism for a user-level threading system
---

# Lab 6 Uthread: switching between threads

## Task 

In this exercise you will design the context switch mechanism for a user-level threading system, and then implement it.

Your job is to come up with a plan to create threads and save/restore registers to switch between threads, and implement that plan.

[Lab: user-level threads and alarm](https://pdos.csail.mit.edu/6.828/2019/labs/syscall.html)

#### Expected Result

```text
~/classes/6828/xv6$ **make qemu**
…
$ **uthread**
thread_a started
thread_b started
thread_c started
thread_c 0
thread_a 0
thread_b 0
thread_c 1
thread_a 1
thread_b 1
…
thread_c 99
thread_a 99
thread_b 99
thread_c: exit after 100
thread_a: exit after 100
thread_b: exit after 100
thread_schedule: no runnable threads
```

### Hints

complete thread\_create to create a properly initialized thread so that when the scheduler switches to that thread for the first time decide where to save/restore registers. You can add fields to struct thread into which to save registers.

## Solution

### Save registers info in `thread`

```c
// Saved registers for kernel context switches.
struct context {
  uint64 ra;
  uint64 sp;

  *// callee-saved*
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
  *// uint64 fs0;*
  *// uint64 fs1;*
  *// uint64 fs2;*
  *// uint64 fs3;*
};
```

### Add context to thread.

```text
struct thread {
  char       stack[STACK_SIZE]; */* the thread's stack */*
  int        state;             */* FREE, RUNNING, RUNNABLE */*
  struct context context;      *// swtch() here to run process*
};
```

### Setup jump function and stack in thread creation

```c
void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  // YOUR CODE HERE
  memset(&t->context, 0, sizeof t->context);
  t->context.ra = (uint64)func;

  t->context.sp = (uint64)(t->stack + STACK_SIZE); 
}
```

Note:

* Set up `ra` return address register to point to the thread function, so switch thread will jump to the function.
* In the standard RISC-V calling convention, the stack grows downward and the stack pointer is always kept 16-byte aligned.
* Set stack pointer to stack top.

### Implement context switch

```c
    .text

/* Switch from current_thread to next thread_thread, and make
 * next_thread the current_thread.  Use t0 as a temporary register,
 * which should be caller saved. */

/*
Caller-saved registers (AKA volatile registers, or call-clobbered) 
are used to hold temporary quantities that need not be preserved across calls.

Callee-saved registers (AKA non-volatile registers, or call-preserved) are 
used to hold long-lived values that should be preserved across calls.
*/

    .globl thread_switch
thread_switch:
    /* YOUR CODE HERE */
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)

    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)

    ret    /* return to ra */
```

We save `ra`, `sp`, and all callee saved registers. This assembly function is called be `thread scheduler`, to switch from current thread to next runnable one. This function saves registers info in thread-&gt;context, and load registers from new thread’s context, then jump executing the new thread.

### Complete thread scheduler

**Define struct and vars**

```c
struct thread {
  char       stack[STACK_SIZE]; /* the thread's stack */
  int        state;             /* FREE, RUNNING, RUNNABLE */
  struct context context;      // swtch() here to run process
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(struct context*, struct context*);
```

**Find next runnable thread**

```c
  struct thread *t, *next_thread;
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }
```

Try to find next one, if reaches the end then start from beginning. If none of them is available, current\_thread is unchanged and resume.

**Switch thread**

```c
if (current_thread != next_thread) {
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    // Invoke thread_switch to switch from t to next_thread:
    thread_switch(&t->context, &current_thread->context);
  }
```

## Ref: Full Uthread Package

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

*/* Possible states of a thread: */*
#define FREE        0x0
#define RUNNING     0x1
#define RUNNABLE    0x2

#define STACK_SIZE  8192
#define MAX_THREAD  4

*// Saved registers for kernel context switches.*
struct context {
  uint64 ra;
  uint64 sp;

  *// callee-saved*
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};

struct thread {
  char       stack[STACK_SIZE]; */* the thread's stack */*
  int        state;             */* FREE, RUNNING, RUNNABLE */*
  struct context context;      *// swtch() here to run process*
};
struct thread all_thread[MAX_THREAD];
struct thread *current_thread;
extern void thread_switch(struct context*, struct context*);

void 
thread_init(void)
{
  *// main() is thread 0, which will make the first invocation to*
  *// thread_schedule().  it needs a stack so that the first thread_switch() can*
  *// save thread 0's state.  thread_schedule() won't run the main thread ever*
  *// again, because its state is set to RUNNING, and thread_schedule() selects*
  *// a RUNNABLE thread.*
  current_thread = &all_thread[0];
  current_thread->state = RUNNING;
}

void 
thread_schedule(void)
{
  struct thread *t, *next_thread;

  */* Find another runnable thread. */*
  next_thread = 0;
  t = current_thread + 1;
  for(int i = 0; i < MAX_THREAD; i++){
    if(t >= all_thread + MAX_THREAD)
      t = all_thread;
    if(t->state == RUNNABLE) {
      next_thread = t;
      break;
    }
    t = t + 1;
  }

  if (next_thread == 0) {
    printf("thread_schedule: no runnable threads\n");
    exit(-1);
  }

  if (current_thread != next_thread) {         */* switch threads?  */*
    next_thread->state = RUNNING;
    t = current_thread;
    current_thread = next_thread;
    */* YOUR CODE HERE*
** Invoke thread_switch to switch from t to next_thread:*
**/*
    thread_switch(&t->context, &current_thread->context);
  } else
    next_thread = 0;
}

void 
thread_create(void (*func)())
{
  struct thread *t;

  for (t = all_thread; t < all_thread + MAX_THREAD; t++) {
    if (t->state == FREE) break;
  }
  t->state = RUNNABLE;
  *// YOUR CODE HERE*
  memset(&t->context, 0, sizeof t->context);
  t->context.ra = (uint64)func;
  */**
*In the standard RISC-V calling convention,*
*the stack grows downward and the stack pointer is*
*always kept 16-byte aligned.*
**/*
  t->context.sp = (uint64)(t->stack + STACK_SIZE); *// how does stack grow?*
}

void 
thread_yield(void)
{
  current_thread->state = RUNNABLE;
  thread_schedule();
}

volatile int a_started, b_started, c_started;
volatile int a_n, b_n, c_n;

void 
thread_a(void)
{
  int i;
  printf("thread_a started\n");
  a_started = 1;
  while(b_started == 0 || c_started == 0)
    thread_yield();

  for (i = 0; i < 100; i++) {
    printf("thread_a %d\n", i);
    a_n += 1;
    thread_yield();
  }
  printf("thread_a: exit after %d\n", a_n);

  current_thread->state = FREE;
  thread_schedule();
}

void 
thread_b(void)
{
  int i;
  printf("thread_b started\n");
  b_started = 1;
  while(a_started == 0 || c_started == 0)
    thread_yield();

  for (i = 0; i < 100; i++) {
    printf("thread_b %d\n", i);
    b_n += 1;
    thread_yield();
  }
  printf("thread_b: exit after %d\n”, b_n);

  current_thread->state = FREE;
  thread_schedule();
}

void 
thread_c(void)
{
  int i;
  printf(“thread_c started\n”);
  c_started = 1;
  while(a_started == 0 || b_started == 0)
    thread_yield();

  for (i = 0; i < 100; i++) {
    printf(“thread_c %d\n”, i);
    c_n += 1;
    thread_yield();
  }
  printf("thread_c: exit after %d\n", c_n);

  current_thread->state = FREE;
  thread_schedule();
}

int 
main(int *argc*, char **argv*[]) 
{
  a_started = b_started = c_started = 0;
  a_n = b_n = c_n = 0;
  thread_init();
  thread_create(thread_a);
  thread_create(thread_b);
  thread_create(thread_c);
  thread_schedule();
  exit(0);
}
```

## 心得

过去五年一直不理解的coroutine现在顿悟了，x86的繁琐和陈旧被risc-v的精简直接击垮。纤程的本质就是context switch，将储存器存入stack, 将另一个纤程的储存器从它的stack读出，将program counter设置好，这，就是fiber.

