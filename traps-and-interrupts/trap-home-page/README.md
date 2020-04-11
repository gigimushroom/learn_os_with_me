# Trap Home Page

## RISC-V trap machinery

\[\[What is trampoline\]\] \[\[What is trapframe\]\]

## Traps from Kernel Space

Xv6 sets a CPU’s `stvec` to `kernelvec` when that CPU enters the kernel from user space.

There is a small windows, that `stvec` hasn’t been set. It is critical that no device interrupt happen during that time.

RISC-V always disables interrupts when it starts to take a trap, and xv6 doesn’t enable them again until after it sets stvec.

```c
void usertrap(void)
{
  // send interrupts and exceptions to kerneltrap(),
  // since we’re now in the kernel.
  w_stvec((uint64)kernelvec);

  // an interrupt will change sstatus &c registers,
  // so don’t enable until done with those registers.
  intr_on();
```

## Traps from User Space

\[\[how system calls get into / out of the kernel\]\]

## How `exec` works

\[\[How exec\(\) works\]\]

### Code

```c
uint64
sys_exec(void)
{
  char path[MAXPATH], *argv[MAXARG];
  int I;
  uint64 uargv, uarg;

  if(argstr(0, path, MAXPATH) < 0 || argaddr(1, &uargv) < 0){
    return -1;
  }
  memset(argv, 0, sizeof(argv));
  for(I=0;; I++){
    if(i >= NELEM(argv)){
      goto bad;
    }
    if(fetchaddr(uargv+sizeof(uint64)*I, (uint64*)&uarg) < 0){
      goto bad;
    }
    if(uarg == 0){
      argv[I] = 0;
      break;
    }
    argv[I] = kalloc();
    if(argv[i] == 0)
      panic("sys_exec kalloc");
    if(fetchstr(uarg, argv[i], PGSIZE) < 0){
      goto bad;
    }
  }

  int ret = exec(path, argv);
```

The `argv` is a pointer to list of arguments from user space. \[image:4B4D1CD2-3D50-48D7-B499-6A1663C8E796-587-0001B03AC2C2ED9E/Screen Shot 2020-03-18 at 5.04.09 PM.png\] Each pointer in RISC-V is 64 bits.

`argaddr(1, &uargv)` is to get the base pointer. Then, do `fetchaddr(uargv+sizeof(uint64)*I, (uint64*)&uarg)` in a loop to find each argument’s address.

`fetchstr(uarg, argv[i], PGSIZE)` is to load the string in `argv[I]`.

### Helpers

```c
// Fetch the uint64 at addr from the current process.
int
fetchaddr(uint64 addr, uint64 *ip)
{
  struct proc *p = myproc();
  if(addr >= p->sz || addr+sizeof(uint64) > p->sz)
    return -1;
  if(copyin(p->pagetable, (char *)ip, addr, sizeof(*ip)) != 0)
    return -1;
  return 0;
}
```

```c
// Fetch the nul-terminated string at addr from the current process.
// Returns length of string, not including nul, or -1 for error.
int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  int err = copyinstr(p->pagetable, buf, addr, max);
  if(err < 0)
    return err;
  return strlen(buf);
}
```

`copyinstr` Copy a null-terminated string from user to kernel.

## How fork works

\[\[How fork\(\) works\]\]

