# XV6 CPU Scheduling

## Why we design scheduling? 

To time-share the CPUs among the processes.

## How xv6 achieves multiplexing?

Xv6 multiplexes by switching each CPU from one process to another in 2 situations: 1. Sleep and wake mechanism 2. Timer fired for a process running for long periods

This multiplexing creates an illusion that each process has its own CPU!

## How kernel does the context switch visualized?

![](../.gitbook/assets/image%20%282%29.png)

## How to implement context switch?

#### Preparation:

Xv6 has a scheduler running in a dedicated thread per CPU. We define a struct to contain all important contents, to save all registers for a process.

```text
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
```

CPU saves the scheduler process’s context. Each process saves context in its state.

```c
*// Per-process state*
struct proc {
  struct spinlock lock;
…

*// these are private to the process, so p->lock need not be held.*
  uint64 kstack;*// Bottom of kernel stack for this process*
  uint64 sz;*// Size of process memory (bytes)*
  pagetable_t pagetable;*// Page table*
  struct trapframe *tf;*// data page for trampoline.S*
  struct context context;*// swtch() here to run process*
…
};
```

A context switch assembly function to save current registers in old. Load from new

```text
swtch:
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

        ret
```

‘ra’ is return address register. ‘sp’ is stack pointer. Follow by a bunch of callee saved register in RISC-V. The last assembly is 'ret'. when swtch returns, it returns to the instructions pointed to by the restored ra register

#### Run time:

1. Process A calling yield\(\) to give up CPU.
2. yield\(\) calls sched\(\), which internally calls the context switch function above.
3. The switch assembly func saves the A’s context on A’s stack. **\(I thought in kernel’s process stack?\)** Yes, the process state are all saved in kernel process stack.
4. Switch restores scheduler context, and jump immediately to scheduler last saved checkpoint. It does not return back to A.
5. Scheduler is going to run next available process. 
6. Eventually scheduler will resume process A, and calling swtch\(\) to resume where process A left before.

## Let’s take a closer look to see How scheduler works

Any process is willing to give up Cpu must do the following: 1. Acquire its own process lock. 2. Release any other locks its holding 3. Update its own state 4. Call sched\(\) C function. Yield\(\), Sleep\(\), exit\(\) all follows this conversion.

Scheduler code

```c
*// Per-CPU process scheduler.*
*// Each CPU calls scheduler() after setting itself up.*
*// Scheduler never returns.  It loops, doing:*
*//  - choose a process to run.*
*//  - swtch to start running that process.*
*//  - eventually that process transfers control*
*//    via swtch back to the scheduler.*
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    *// Avoid deadlock by ensuring that devices can interrupt.*
    intr_on();
    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        *// Switch to chosen process.  It is the process's job*
        *// to release its lock and then reacquire it*
        *// before jumping back to us.*
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->scheduler, &p->context);

        *// Process is done running for now.*
        *// It should have changed its p->state before coming back.*
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }
    if(found == 0){
      intr_on();
      asm volatile("wfi");
    }
  }
}
```

A non-stop for loop tries to find next RUNNABLE process. Note: As soon as another process switching back to scheduler, it resumes at the code line after ‘swtch\(\)’. It will do ‘c-&gt;proc = 0’

One crucial part is lock. Scheduler will acquire a lock, then do context switch. The resumed process will release the lock. If process wants to give up CPU, it needs to acquire the lock, then scheduler is going to release the lock. The above is different than normal lock/unlock conventions. But it has to be designed this way to make context switch critical section code protected!

There are 2 **invariants**: 1. If a process is running, a timer can safely switch away it. The CPU registers must hold process’s register values, and c-&gt;proc refer to it. 2. If a process is runnable, its p-&gt;context must hold its registers, no CPU is executing on the process’s kernel stack, no CPU references to it. **If above invariant is not true, that’s the place we need a lock.**

Enable/Disable interrupts Values of cpuid and mycpu are fragile: if the timer were to interrupt and cause the thread to yield and then move to a different CPU, a previously returned value would no longer be correct. To avoid this problem, xv6 **requires that callers disable interrupts, and only enable them after they finish using the returned struct cpu.**

Similarly, `myproc()`disable interrupts, and re-enable after invokes `mycpu()`

```text
struct proc*
myproc(void) {
  push_off();
  struct cpu *c = mycpu();
  struct proc *p = c->proc;
  pop_off();
  return p;
}
```

The return value of myproc is safe to use even if interrupts are enabled: if a timer interrupt moves the calling process to a different CPU, its struct proc pointer will stay the same.

### Graph

![](../.gitbook/assets/image%20%2812%29.png)

## How process intentionally interact with each other?

Use sleep and wakeup.

### How it is used?

```c
Void V(struct semaphore *s) 
{
  acquire(&s->lock);
  S->count += 1;
  wakup(s);
  release($s->lock);
}
Void P(struct semaphore *s)
{
  while(s->count == 0)
    sleep(s, &s->lock);
  S->count == 1;
  release(&s->lock);
}
```

Note: we need `sleep` to atomically release `s->lock` and put the consuming process to sleep. Before process waking up, `sleep` needs to acquire the `s->lock` again.

#### Sleep and wakeup implementation

```c
*// Atomically release lock and sleep on chan.*
*// Reacquires lock when awakened.*
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();

  *// Must acquire p->lock in order to*
  *// change p->state and then call sched.*
  *// Once we hold p->lock, we can be*
  *// guaranteed that we won’t miss any wakeup*
  *// (wakeup locks p->lock),*
  *// so it’s okay to release lk.*
  if(lk != &p->lock){  *//DOC: sleeplock0*
    acquire(&p->lock);  *//DOC: sleeplock1*
    release(lk);
  }

  *// Go to sleep.*
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  *// Tidy up.*
  p->chan = 0;

  *// Reacquire original lock.*
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}

*// Wake up all processes sleeping on chan.*
*// Must be called without any p->lock.*
void
wakeup(void *chan)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++) {
    acquire(&p->lock);
    if(p->state == SLEEPING && p->chan == chan) {
      p->state = RUNNABLE;
    }
    release(&p->lock);
  }
}
```

A few things:

* Before process sleeping, it must hold the lk lock, so no wake up is lost.
* Before process sleeping, it must hold the process lock, so it can do context switch to scheduler.
* Scheduler will release the process lock.
* When the sleeping process is resumed, it must release the process lock, previously locked by scheduler.
* The resuming process must acquire the lk lock. Since it needs to consume the data without any interruption.



