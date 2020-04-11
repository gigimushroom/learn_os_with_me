# XV6 Device Driver

## system/kernel/hardware

## Why we have a Driver?

A driver is the code in an operating system that manages a particular device: 1. It tells the device hardware to perform operations,. 2. Configures the device to generate interrupts when done. 3. Handles the resulting interrupts,. 4. Interacts with processes that may be waiting for I/O from the device.

## Initialization

### Config Hardware

Xv6’s main calls `consoleinit` \(kernel/console.c:189\) to initialize the UART hardware, and to configure the UART hardware to generate **input** interrupts \(`kernel/uart.c:34`\).

```c
void
consoleinit(void)
{
  initlock(&cons.lock, “cons”);

  uartinit();

  // connect read and write system calls
  // to consoleread and consolewrite.
  devsw[CONSOLE].read = consoleread;
  devsw[CONSOLE].write = consolewrite;
}
```

### Create console device

In `init.c`, create a console device:

```text
mknod(“console”, 1, 1);
open(“console”, O_RDWR);
```

The file path is `console`, type is `T_DEVICE`.

`mknod` implementation.

```c
uint64
sys_mknod(void)
{
  struct inode *ip;
  char path[MAXPATH];
  int major, minor;

  begin_op(ROOTDEV);
  if((argstr(0, path, MAXPATH)) < 0 ||
     argint(1, &major) < 0 ||
     argint(2, &minor) < 0 ||
     (ip = create(path, T_DEVICE, major, minor)) == 0){
    end_op(ROOTDEV);
    return -1;
  }
```

At this point, we have a console device. A file descriptor 0 points to this device.

## How to read or write from device

Let’s look at `fileread`

```c
int
fileread(struct file *f, uint64 addr, int n)
{
  int r = 0;

  if(f->readable == 0)
    return -1;

  if(f->type == FD_PIPE){
    r = piperead(f->pipe, addr, n);
  } else if(f->type == FD_DEVICE){
    if(f->major < 0 || f->major >= NDEV || !devsw[f->major].read)
      return -1;
    r = devsw[f->major].read(f, 1, addr, n);
```

If the file type is DEVICE, we call the special `read` function to perform read operation. Note: The special read is set up in `consoleinit`.

`consoleread` sleeps and wait for a line of inputs, then copy to user space. Details see flow section.

## Flow

### Every input char generates an interrupt

When the user types a character, the UART hardware asks the RISC-V to raise an interrupt.

### Trap handler determines interrupt source

xv6’s trap handling code calls `devintr` \(kernel/trap.c:177\). `devintr` looks at the RISC-V `scause` register to discover that the interrupt is from an external device. Then it asks a hardware unit called the PLIC \[1\] to tell it which device interrupted \(kernel/trap.c:186\). If it was the UART, `devintr`calls `uartintr`.

### Accumulate inputs and Wake up

#### Read one char and hand over

`uartintr` \(kernel/uart.c:84\) reads any waiting input characters from the UART hardware and hands them to `consoleintr`.

```c
// trap.c calls here when the uart interrupts.
void
uartintr(void)
{
  while(1){
    int c = uartgetc();
    if(c == -1)
      break;
    consoleintr(c);
  }
}
```

#### Accumulate inputs

The job of `consoleintr` is to accumulate input characters in `cons.buf` until a whole line arrives.

#### Wake up when get a whole line

When a newline arrives, `consoleintr` wakes up a waiting `consoleread` \(if there is one\).

#### Copy to user space

Once woken, `consoleread` will observe a full line in `cons.buf`, copy it to user space and return to user space.

## Question:

#### How does `open(“console", O_RDWR)` opens a file with DEVICE type?

Research: It seems this `console` string is defined somewhere. I tried to change to other string, and it does not work. Using this `console` string to open file, the file `inode` returns a file with type is DEVICE.

**Answer:**

OS create a `console` device using system call `mknod`. Any read or write with `console` is interacting with the UART hardware.

