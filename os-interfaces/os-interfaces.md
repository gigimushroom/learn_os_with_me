# OS interfaces

#### The job of OS is

1. Share programs.
2. Abstract low-level hardware, so program need not concern what disk is being used.
3. Provide ways for programs to interact.

#### Design philosophy

OS defines interfaces to be simple and narrow. Combine programs to provide much generality.

#### User space and Kernel Space

![](../.gitbook/assets/image%20%282%29.png)

When a process needs to invoke a kernel service, it invokes a system call. The system call enters the kernel, the kernel performs the service and returns.

#### Privileges

Kernel uses CPU’s hardware protection mechanisms to ensure each process in user space can only access its own memory. When a user program invokes a system call, the hardware raises the privilege level and starts executing a pre-arranged function in kernel.

#### The shell

The shell is an ordinary program that reads commands from the user and executes them. In our journey, we will be implement a new shell ourself \(see lab2\).

#### Xv6 system calls

![](../.gitbook/assets/image%20%2816%29.png)

