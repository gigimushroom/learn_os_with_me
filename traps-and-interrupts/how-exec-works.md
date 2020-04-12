---
description: Split the flow into 2 phases.
---

# How exec\(\) works

## How exec\(\) works

Split the flow into 4 phases: 1. Read args. 2. Load program from disk to memory. 3. Allocate stacks. 4. Prepare the new images.

### Read User Args

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

The `argv` is a pointer to list of arguments from user space. 

![](../.gitbook/assets/image%20%2810%29.png)

Each pointer in RISC-V is 64 bits.

`argaddr(1, &uargv)` is to get the base pointer. Then, do `fetchaddr(uargv+sizeof(uint64)*I, (uint64*)&uarg)` in a loop to find each argument’s address.

`fetchstr(uarg, argv[i], PGSIZE)` is to load the string in `argv[I]`.

#### Helper

Helpers

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

### Load program and Allocate pages

1. Check ELF header
2. Load program into memory
3. Allocate 2 pages, one for user stack. Another for guard page.

![](../.gitbook/assets/image%20%2827%29.png)

Note: this graph has a bug. The following 3 items should not exist: `argv, argc, 0xFFFFFF`

### Prepare the new image

Push argument strings to stack. Prepare rest of stack.

Push the array of `argv[]` pointers to stack.

Set `argv` in a1 register. Set process info, including `pagetable`, `sz` Set `epc = elf.entry`, initial program counter to main initial stack pointer. return `argc`, in RSIC-V, this sets `argc` in register a0. So arguments `main(argc, argv)` is set.

The jump preparation is done. Program will go to `main`, and get args set properly.

#### Core `exec` code for reference

```c
int
exec(char *path, char **argv)
{
  char *s, *last;
  int i, off;
  uint64 argc, sz, sp, ustack[MAXARG+1], stackbase;
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pagetable_t pagetable = 0, oldpagetable;
  struct proc *p = myproc();

  begin_op(ROOTDEV);

  if((ip = namei(path)) == 0){
    end_op(ROOTDEV);
    return -1;
  }
  ilock(ip);

  // Check ELF header
  if(readi(ip, 0, (uint64)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pagetable = proc_pagetable(p)) == 0)
    goto bad;

  // Load program into memory.
  sz = 0;
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, 0, (uint64)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    if((sz = uvmalloc(pagetable, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op(ROOTDEV);
  ip = 0;

  p = myproc();
  uint64 oldsz = p->sz;

  // Allocate two pages at the next page boundary.
  // Use the second as the user stack.
  sz = PGROUNDUP(sz);
  if((sz = uvmalloc(pagetable, sz, sz + 2*PGSIZE)) == 0)
    goto bad;
  uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;

  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp -= strlen(argv[argc]) + 1;
    sp -= sp % 16; // riscv sp must be 16-byte aligned
    if(sp < stackbase)
      goto bad;
    if(copyout(pagetable, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[argc] = sp;
  }
  ustack[argc] = 0;

  // push the array of argv[] pointers.
  sp -= (argc+1) * sizeof(uint64);
  sp -= sp % 16;
  if(sp < stackbase)
    goto bad;
  if(copyout(pagetable, sp, (char *)ustack, (argc+1)*sizeof(uint64)) < 0)
    goto bad;

  // arguments to user main(argc, argv)
  // argc is returned via the system call return
  // value, which goes in a0.
  p->tf->a1 = sp;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(p->name, last, sizeof(p->name));

  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
```

#### TODO

understand ELF.

