# Process and Memory

Xv6 time-share processes. Save its CPU registers when process not running, and restoring them back when it resumes. Process can create another process using `fork`

`wait` returns the pid of the exited child of the current process. `wait` waits until one child to exit.

Parent process and child process are executing with different memory and different registers. Changing a value does not affect another.

`exec` system call replaces the calling process’ memory with a new memory image loaded from a file. The file is ELF format.

Xv6 Shell calls `getcmd` to reads a line of user input, then call `fork`, which creates a copy of the shell process. The parents calls `wait`, while the child is running the command.

Process wants more memory could call `sbrk(n)`.

### Questions: Why `fork` and `exec` not combine together?

#### Answer:

If they are separate, the shell can fork a child, use open, close, dup in the child to change the standard input and output file descriptors, and then exec! Executed program does not need to understand how to redirect I/O. It is simple.

