# Locking in Xv6

## Using lock in XV6

![](../.gitbook/assets/image%20%2845%29.png)

Must have a global deadlock-avoiding lock order if holding multiple locks together.

## Locks and interrupt handlers

The interaction of spinlocks and interrupts raises a potential danger.

Suppose `sys_sleep` holds `tickslock`, and its CPU is interrupted by a timer interrupt. `clockintr` would try to acquire `tickslock`, see it was held, and wait for it to be released.

In this situation, `tickslock` will never be released: only `sys_sleep` can release it, but sys\_sleep will not continue running until `clockintr` returns. So the CPU will deadlock!

### Solution

To avoid this situation, if a spinlock is used by an interrupt handler, a CPU must never hold that lock with interrupts enabled.

XV6 spin lock disable interrupts in `acquire`, and at end of `release`.

```c
// Acquire the lock.
// Loops (spins) until the lock is acquired.
void
acquire(struct spinlock *lk)
{
  push_off(); // disable interrupts to avoid deadlock.
  if(holding(lk))
    panic(“acquire”);

  __sync_fetch_and_add(&(lk->n), 1);

  // On RISC-V, sync_lock_test_and_set turns into an atomic swap:
  //   a5 = 1
  //   s1 = &lk->locked
  //   amoswap.w.aq a5, a5, (s1)
  while(__sync_lock_test_and_set(&lk->locked, 1) != 0) {
     __sync_fetch_and_add(&lk->nts, 1);
  }

  // Tell the C compiler and the processor to not move loads or stores
  // past this point, to ensure that the critical section’s memory
  // references happen strictly after the lock is acquired.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Record info about lock acquisition for holding() and debugging.
  lk->cpu = mycpu();
}
```

```c
// Release the lock.
void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->cpu = 0;

  // Tell the C compiler and the CPU to not move loads or stores
  // past this point, to ensure that all the stores in the critical
  // section are visible to other CPUs before the lock is released,
  // and that loads in the critical section occur strictly before
  // the lock is released.
  // On RISC-V, this emits a fence instruction.
  __sync_synchronize();

  // Release the lock, equivalent to lk->locked = 0.
  // This code doesn't use a C assignment, since the C standard
  // implies that an assignment might be implemented with
  // multiple store instructions.
  // On RISC-V, sync_lock_release turns into an atomic swap:
  //   s1 = &lk->locked
  //   amoswap.w zero, zero, (s1)
  __sync_lock_release(&lk->locked);

  pop_off();
}
```

XV6 also does book-keeping to cope with nested critical sections. `acquire` calls `push_off (kernel/spinlock.c:87)` and release calls `pop_off (kernel/spinlock.c:98)` to track the nesting level of locks on the current CPU.

Code for push and pop

```c
// push_off/pop_off are like intr_off()/intr_on() except that they are matched:
// it takes two pop_off()s to undo two push_off()s.  Also, if interrupts
// are initially off, then push_off, pop_off leaves them off.

void
push_off(void)
{
  int old = intr_get();
  if(old)
    intr_off();
  if(mycpu()->noff == 0)
    mycpu()->intena = old;
  mycpu()->noff += 1;
}

void
pop_off(void)
{
  if(intr_get())
    panic(“pop_off - interruptible");
  struct cpu *c = mycpu();
  if(c->noff < 1)
    panic(“pop_off”);
  c->noff -= 1;
  if(c->noff == 0 && c->intena)
    intr_on();
}
```

## Instruction and memory ordering

Many compilers and CPUs, however, execute code out of order to achieve higher performance.

Compilers and CPUs follow rules when they re-order to ensure that they don’t change the results of correctly-written **serial code**. However, the rules do allow **re-ordering that changes the results of concurrent code**, and can easily lead to incorrect behavior on multiprocessors

xv6 uses `__sync_synchronize()` in spin lock `acquire` and `release`, which is a memory barrier: it tells the compiler and CPU to not reorder loads or stores across the barrier.

## Sleep lock

As we know, yielding while holding a spinlock is illegal because it might lead to deadlock if a second thread then tried to acquire the spinlock.

Use sleep lock: a sleep-lock has a locked field that is protected by a spinlock, and `acquiresleep` ’s call to sleep atomically yields the CPU and releases the spinlock. The result is that other threads can execute while `acquiresleep` waits.

```c
// Long-term locks for processes
struct sleeplock {
  uint locked;       // Is the lock held?
  struct spinlock lk; // spinlock protecting this sleep lock

  // For debugging:
  char *name;        // Name of lock.
  int pid;           // Process holding lock
};
```

```c
void
initsleeplock(struct sleeplock *lk, char *name)
{
  initlock(&lk->lk, “sleep lock”);
  lk->name = name;
  lk->locked = 0;
  lk->pid = 0;
}

void
acquiresleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  while (lk->locked) {
    sleep(lk, &lk->lk);
  }
  lk->locked = 1;
  lk->pid = myproc()->pid;
  release(&lk->lk);
}

void
releasesleep(struct sleeplock *lk)
{
  acquire(&lk->lk);
  lk->locked = 0;
  lk->pid = 0;
  wakeup(lk);
  release(&lk->lk);
}
```

```c
// Atomically release lock and sleep on chan.
// Reacquires lock when awakened.
void
sleep(void *chan, struct spinlock *lk)
{
  struct proc *p = myproc();

  // Must acquire p->lock in order to
  // change p->state and then call sched.
  // Once we hold p->lock, we can be
  // guaranteed that we won’t miss any wakeup
  // (wakeup locks p->lock),
  // so it's okay to release lk.
  if(lk != &p->lock){  //DOC: sleeplock0
    acquire(&p->lock);  //DOC: sleeplock1
    release(lk);
  }

  // Go to sleep.
  p->chan = chan;
  p->state = SLEEPING;

  sched();

  // Tidy up.
  p->chan = 0;

  // Reacquire original lock.
  if(lk != &p->lock){
    release(&p->lock);
    acquire(lk);
  }
}

// Wake up all processes sleeping on chan.
// Must be called without any p->lock.
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

## Summary

Spin-locks are best suited to short critical sections, since waiting for them wastes CPU time; sleep-locks work well for lengthy operations.

