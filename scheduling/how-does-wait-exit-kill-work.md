# How does wait\(\), exit\(\), kill\(\) work?

`wait()` is waiting for a child process to exist, and return its pid. It checks for any process's parent is itself. If the process state is ZOMBIE, copy some data, do the clean up. If has children but not dead yet, the parent is going to sleep, inside a big for loop.

`exit()` first close all open files. Acquire parent lock, and child itself’s lock. Give my children to the root init process. Wake up my parent. Make me as ZOMBIE state. Release parent lock. Jump into scheduler and never returns.

`kill()` does very little. It just set killed flag to 1. If state is `SLEEPING`, change it to `RUNNABLE`. So the process is waken up. When the process entering or leaving kernel, the trap will find out it is killed, and call `exit()`. Since xv6 using loops for sleep, if the killed process is wakeup from sleeping, the while condition still not applied, so it is sleeping again. But this is a problem, since it requires a lot of places handling `killed == 1` correctly.

### Challenges

1. Implement signal in xv6 

2. Implement semaphore in xv6 

3. Research how real OS use explicit free list to find free proc structures, instead of searching in `allocproc` in linear-time.

