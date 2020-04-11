---
description: Implement network sockets
---

# Lab 10 Networking Part 2

## Network Sockets

Your job is to implement the missing functionality necessary to support network sockets. This includes adding and integrating functions to support reading, writing, and closing sockets. It also includes completing the implementation of `sockrecvudp()`, which is called each time a new UDP packet is received. To achieve this, fill in the missing sections in kernel/`sysnet.c` and modify `kernel/file.c` to call your socket methods. [Lab: networking](https://pdos.csail.mit.edu/6.828/2019/labs/net.html)

#### Expectation

When you are finished, run the test program. If everything is correct, you will get the following output: \(on the host in one terminal\)

```text
$ make server
python2 server.py 26099
listening on localhost port 26099
(then on xv6 in another terminal on the same machine run
  nettests; see below)
hello world!
```

## Solution

### How it works end to end

#### How `sys_connect`, and `sys_write` work?

![](../.gitbook/assets/image%20%2811%29.png)

#### How receiving a packet work end to end?

\(from E1000 hardware, to trap handler, to user space\)

![](../.gitbook/assets/image%20%2813%29.png)

#### How reading a socket works?

![](../.gitbook/assets/image%20%2818%29.png)

### Code

#### Implement socket read

```c
int sockread(struct file *f, uint64 addr, int n){
  // check if rxq is empty using mbufq_empty(), 
  // and if it is, use sleep() to wait until an mbuf is enqueued.
  //printf(“sock read size %d\n”, n);
  acquire(&f->sock->lock);
  while (mbufq_empty(&f->sock->rxq)) {
    sleep(&f->sock->rxq, &f->sock->lock);
  }
  // Using mbufq_pophead, pop the mbuf from rxq and use copyout() 
  // to move its payload into user memory. 
  struct mbuf * m = mbufq_pophead(&f->sock->rxq);
  if (!m) {
    printf(“what the heck\n”);
    return -1;
  }
  struct proc* p = myproc();
  int len = n > m->len ? m->len : n;
  //printf(“sock read size %d IS WAKEN UP! this buffer is size %d\n", n, m->len);
  if (copyout(p->pagetable, addr, m->head, len) == -1) {
    release(&f->sock->lock);
    mbuffree(m);
    return -1;
  }

  release(&f->sock->lock);
  // Free the mbuf using mbuffree() to finish
  mbuffree(m);
  //printf(“sock read is finished with size %d\n”, len);
  return len;
}
```

#### Implement socket write

```c
int sockwrite(struct file *f, uint64 addr, int n) {
  // allocate a new mbuf, taking care to leave enough 
  // headroom for the UDP, IP, and Ethernet headers.
  //printf(“sock write size %d\n”, n);
  int head_size = sizeof(struct udp) + sizeof(struct ip) + sizeof(struct eth);
  struct mbuf *m = mbufalloc(head_size);

  mbufput(m, n);
  struct proc* p = myproc();
  // Use mbufput() and copyin() to transfer the payload from user memory into the mbuf.
  if (copyin(p->pagetable, m->head, addr, n) == -1) {
    mbuffree(m);
    return -1;
  }

  // send the mbuf.
  struct sock * s = f->sock;
  net_tx_udp(m, s->raddr, s->lport, s->rport);
  return n;
}
```

#### Implement socket close

```c
void sockclose(struct file *f) {
  // remove the socket from the sockets list. 
  struct sock *s = f->sock;
  struct sock *p = sockets;
  if (s==p) {
    sockets = 0;
  } else {
    //printf(“TODO: more sokcets in list.\n”);
  }

  // Then, free the socket object. Be careful to free any mbufs 
  // that have not been read first, before freeing the struct sock.
  if (mbufq_empty(&s->rxq)) {
    struct mbuf *m = mbufq_pophead(&s->rxq);
    while(m) {
      mbuffree(m);
    }
  }
  kfree(s);
}
```

TODO: finish deleting item from linked list.

#### Implement `sockrecvudp`

This function is called by protocol handler layer to deliver UDP packets. What it does: 1. Find the socket that handles this mbuf and deliver it. 2. Waking any sleeping reader. 3. Free the mbuf if there are no sockets registered to handle it.

```c
void
sockrecvudp(struct mbuf *m, uint32 raddr, uint16 lport, uint16 rport)
{
  //
  // Your code here.
  //
  // Find the socket that handles this mbuf and deliver it, waking
  // any sleeping reader. Free the mbuf if there are no sockets
  // registered to handle it.
  //
  struct sock *s = sockets;
  acquire(&lock);
  while (s) {
    if (s->raddr == raddr &&
        s->lport == lport &&
          s->rport == rport) {
      break;
    }
    s = s->next;
  }
  release(&lock);

  if (s) {
    acquire(&s->lock);
    mbufq_pushtail(&s->rxq, m);
    release(&s->lock);
    wakeup(&s->rxq);
    return;
  }

  mbuffree(m);
}
```

### Integrate above methods to `file.c`

Define your socket methods for `read, write, and close` in `kernel/defs.h`. Integrate each of these methods into the appropriate call sites in kernel/file.c by checking whether the socket type is `FD_SOCK`.

## How to test 

Borrow from `nettests.c`

```c
//
// send a UDP packet to the localhost (outside of qemu),
// and receive a response.
//
static void
ping(uint16 sport, uint16 dport, int attempts)
{
  int fd;
  char obuf[13] = “hello world!”;
  uint32 dst;

  // 10.0.2.2, which qemu remaps to the external host,
  // i.e. the machine you’re running qemu on.
  dst = (10 << 24) | (0 << 16) | (2 << 8) | (2 << 0);

  // you can send a UDP packet to any Internet address
  // by using a different dst.

  if((fd = connect(dst, sport, dport)) < 0){
    fprintf(2, “ping: connect() failed\n”);
    exit(1);
  }

  for(int i = 0; i < attempts; i++) {
    if(write(fd, obuf, sizeof(obuf)) < 0){
      fprintf(2, “ping: send() failed\n");
      exit(1);
    }
  }

  char ibuf[128];
  int cc = read(fd, ibuf, sizeof(ibuf));
  if(cc < 0){
    fprintf(2, "ping: recv() failed\n");
    exit(1);
  }

  close(fd);
  if (strcmp(obuf, ibuf) || cc != sizeof(obuf)){
    fprintf(2, “ping didn’t receive correct payload\n”);
    exit(1);
  }
}
```

