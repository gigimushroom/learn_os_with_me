# How unix pipes work?

```c
int pipewrite(struct pipe *pi, uint64 addr, int n)
{
  int I;
  char ch;
  struct proc *pr = myproc();

  acquire(&pi->lock);
  for(I = 0; I < n; I++){
    while(pi->nwrite == pi->nread + PIPESIZE){  *//DOC: pipewrite-full*
      if(pi->readopen == 0 || myproc()->killed){
        release(&pi->lock);
        return -1;
      }
      wakeup(&pi->nread);
      sleep(&pi->nwrite, &pi->lock);
    }
    if(copyin(pr->pagetable, &ch, addr + I, 1) == -1)
      break;
    pi->data[pi->nwrite++ % PIPESIZE] = ch;
  }
  wakeup(&pi->nread);
  release(&pi->lock);
  return n;
}

int
piperead(struct pipe *pi, uint64 addr, int n)
{
  int i;
  struct proc *pr = myproc();
  char ch;

  acquire(&pi->lock);
  while(pi->nread == pi->nwrite && pi->writeopen){  *//DOC: pipe-empty*
    if(myproc()->killed){
      release(&pi->lock);
      return -1;
    }
    sleep(&pi->nread, &pi->lock); *//DOC: piperead-sleep*
  }
  for(I = 0; I < n; I++){  *//DOC: piperead-copy*
    if(pi->nread == pi->nwrite)
      break;
    ch = pi->data[pi->nread++ % PIPESIZE];
    if(copyout(pr->pagetable, addr + i, &ch, 1) == -1)
      break;
  }
  wakeup(&pi->nwrite);  *//DOC: piperead-wakeup*
  release(&pi->lock);
  return i;
}
```

* Pipe buffer wraps around. 
* a full buffer `(nwrite == nread+PIPESIZE)`
* an empty buffer`(nwrite == nread)`
* `Pipewrite` wakes up reader if buffer is full, or written is down.
* `Piperead` sleep if no more data. Or read data from pipe, copy out to address. Then wake up the write channel.

### Flow

![](../.gitbook/assets/image%20%2820%29.png)

#### TODO

Add more code.

#### A few open questions:

1. In `piperead`, why we wake up write first, then release `pi->lock`?
2. Why `piperead` does not go back to sleep after reading? Why there is no loop for `piperead`? Or is there an outside loop?

