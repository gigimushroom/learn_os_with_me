# How fork\(\) works

## Flow 

1. Create new process 

2. Copy user memory from parent to child 

3. Set size, and parent. 

4. Save user registers. 

5. Increment reference counts on open file descriptors. 

6. Set pid, change state to runnable 

7. return 0

```c
// Create a new process, copying the parent.
// Sets up child kernel stack to return as if from fork() system call.
int
fork(void)
{
  int i, pid;
  struct proc *np;
  struct proc *p = myproc();

  // Allocate process.
  if((np = allocproc()) == 0){
    return -1;
  }

  // Copy user memory from parent to child.
  if(uvmcopy(p->pagetable, np->pagetable, p->sz) < 0){
    freeproc(np);
    release(&np->lock);
    return -1;
  }
  np->sz = p->sz;

  np->parent = p;

  // copy saved user registers.
  *(np->tf) = *(p->tf);

  // Cause fork to return 0 in the child.
  np->tf->a0 = 0;

  // increment reference counts on open file descriptors.
  for(i = 0; i < NOFILE; i++)
    if(p->ofile[i])
      np->ofile[i] = filedup(p->ofile[i]);
  np->cwd = idup(p->cwd);

  safestrcpy(np->name, p->name, sizeof(p->name));

  pid = np->pid;

  np->state = RUNNABLE;

  release(&np->lock);

  return pid;
}
```

#### Copy memory from parent to child

Given a parent process's page table, copy its memory into a child’s page table. 

Copies both the page table and the physical memory. returns 0 on success, -1 on failure. 

Free any allocated pages on failure.

```c
int
uvmcopy(pagetable_t *old*, pagetable_t *new*, uint64 *sz*)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;

  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic(“uvmcopy: pte should exist”);
    if((*pte & PTE_V) == 0)
      panic(“uvmcopy: page not present”);
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    flags = flags & ~PTE_W;

    // Clear parent and child PTE_W bit.
    *pte |= flags;
    if(mappages(new, i, PGSIZE, (uint64)pa, flags) != 0){
      goto err;
    }
  }
  return 0;

 err:
  uvmunmap(new, 0, i, 1);
  return -1;
}
```

We have the process current size sz, and for each page, we copy to child. 

Each i is the address of each page, calling walk\(\) find out the Page Table Entry\(pte\). 

We can get flags and physical address from PTE. 

Allocate memory for child, copy the content, and flags.

