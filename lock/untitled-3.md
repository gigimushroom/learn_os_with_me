# Hardware Support Locking

## Concept

To evaluate whether a lock works, there are three basic criteria: 1. Mutual exclusion 2. Fairness 3. Performance

## Design and Implement Lock

Locks requires OS help. Building spin locks could use: 

1. `Test-And-Set`. 

2. `Compare-And-Swap`. 

3. `Fetch-And-Add`

XV6 uses option 1.

```c
void lock(lock_t *lock) {
  while (TestAndSet(&lock->flag, 1) == 1) ; 
     // spin-wait (do nothing)
}
```

`TestAndSet` swap flag's value with 1, and return the old value. If result is 1, it was locked before, so we spin and wait. If result is 0, it was unlocked, and we locked succesfully, we can now enter the critical section.

## Case Study: Linux-based Futex Locks

### How it works

Mutex lock counter: bit 31 clear means unlocked; bit 31 set means locked.

All code that looks at bit 31 first increases the ‘number of interested threads’ usage counter, which is in bits 0-30. All negative mutex values indicate that the mutex is still locked.

### Code

This code snippet from `lowlevellock.h` in the nptl library \(part of the gnu libc library\).

```c
void mutex_lock (int *mutex)
{
  unsigned int v;

  /* Bit 31 was clear, we got the mutex.  (this is the fastpath).  */
  if (atomic_bit_test_set (mutex, 31) == 0)
    return;

  atomic_increment (mutex);

  while (1)
    {
      if (atomic_bit_test_set (mutex, 31) == 0)
      {
        atomic_decrement (mutex);
        return;
      }
          /* We have to wait now. First make sure the futex value we are monitoring is truly negative (i.e. locked). */
          v = *mutex;
          if (v >= 0)
            continue;

          lll_futex_wait (mutex, v);
    }
}
```

```c
void mutex_unlock (int *mutex)
{
  /* Adding 0x80000000 to the counter results in 0 if and only if there are not other interested threads - we can return (this is the fastpath).  */
  if (atomic_add_zero (mutex, 0x80000000))
    return;

  /* There are other threads waiting for this mutex, wake one of them up.  */
  lll_futex_wake (mutex, 1);
}
```

### Explain

In `unlock`, `0x80000000` hex is binary: `10 00000 00000 00000 00000 00000 00000` \(31 zeros\).

`atomic_add_zero` impl is:

```c
# define atomic_add_zero(mem, value)                \
  ({ __typeof (value) __atg13_value = (value);              \
     atomic_exchange_and_add (mem, __atg13_value) == -__atg13_value; })
```

`exchange_and_add` :Add VALUE to _MEM and return the old value of_ MEM.

What this function does is: Convert hex value to a proper int. Use negative of value. Compare result of exchange\_and\_add to negative of value, as return result.

The flow of 2 cases: 

![](../.gitbook/assets/image%20%2827%29.png)

Case 1: If no one waiting, return T. 

Case 2: If someone still waiting, return False.

