# Table of contents

* [Learn OS with me](README.md)

## OS Interfaces

* [OS interfaces](os-interfaces/os-interfaces.md)
* [I/O and File descriptors](os-interfaces/i-o-and-file-descriptors.md)
* [Process and Memory](os-interfaces/process-and-memory.md)
* [Pipes](os-interfaces/pipes.md)
* [File](os-interfaces/file.md)

## OS Organization

* [OS Organization](os-organization/untitled.md)
* [Challenge yourself](os-organization/challenge-yourself.md)

## Memory Management <a id="virtual-memory"></a>

* [XV6 Virtual Memory](virtual-memory/xv6-virtual-memory.md)
* [MM Q&A](virtual-memory/mm-q-and-a.md)
* [Page Table](virtual-memory/page-table/README.md)
  * [Part 1: How to translate address](virtual-memory/page-table/part-1-how-to-translate-address.md)
  * [Part 2: Create an Address Space](virtual-memory/page-table/part-2-create-an-address-space.md)
  * [Part 3: How Page Table is used](virtual-memory/page-table/part-3-how-page-table-is-used.md)
  * [Part 4: Page Fault and Swap](virtual-memory/page-table/part-4-page-fault-and-swap.md)
  * [Part 5: How to operate on page tables](virtual-memory/page-table/part-5-how-to-operate-on-page-tables.md)
* [xv6 buddy allocator](virtual-memory/untitled-1/README.md)
  * [How to display physical memory](virtual-memory/untitled-1/how-to-display-physical-memory.md)
* [Memory Management Walk Through](virtual-memory/untitled.md)

## Traps and Interrupts

* [Trap Home Page](traps-and-interrupts/trap-home-page.md)
* [What is trapframe](traps-and-interrupts/untitled-2.md)
* [What is trampoline](traps-and-interrupts/untitled-3.md)
* [Traps from kernel space](traps-and-interrupts/traps-from-kernel-space.md)
* [How fork\(\) works](traps-and-interrupts/how-fork-works.md)
* [How system calls get into/out of the kernel](traps-and-interrupts/untitled.md)
* [How exec\(\) works](traps-and-interrupts/how-exec-works.md)

## Scheduling

* [XV6 CPU Scheduling](scheduling/xv6-cpu-scheduling.md)
* [How unix pipes work?](scheduling/how-unix-pipes-work.md)
* [How does wait\(\), exit\(\), kill\(\) work?](scheduling/how-does-wait-exit-kill-work.md)

## File System

* [Overview and Disk Layout](file-system/overview-and-disk-layout.md)
* [Buffer Cache](file-system/buffer-cache.md)
* [Design Inode Layer](file-system/design-inode-layer.md)
* [Inode Content](file-system/inode-content.md)
* [Block Allocator](file-system/block-allocator.md)
* [Design a log system for crash recovery](file-system/design-a-log-system-for-crash-recovery.md)
* [Directory Layer](file-system/directory-layer.md)
* [Path names](file-system/path-names.md)
* [File Descriptor Layer](file-system/untitled-2.md)
* [FS System Calls](file-system/untitled.md)
* [XV6 VS Real World](file-system/xv6-vs-real-world.md)
* [Make Xv6 File disk management system](file-system/make-xv6-file-disk-management-system.md)
* [Write FS simulator in python](file-system/write-fs-simulator-in-python.md)
* [How Redirect Shell command works](file-system/how-redirect-shell-command-works.md)

## Concurrency <a id="lock"></a>

* [Spinlock](lock/untitled.md)
* [How linux select work](lock/how-linux-select-work.md)
* [Hardware Support Locking](lock/untitled-3.md)
* [Exercise: Implement atomic counter](lock/untitled-2.md)
* [Locking in Xv6](lock/locking-in-xv6.md)
* [Concurrency in Xv6](lock/concurrency-in-xv6.md)
* [Exercise: Socket Programming with Event loop](lock/exercise-socket-programming-with-event-loop.md)
* [Untitled](lock/untitled-1.md)

## Labs

* [Lab 1 Xv6 and Unix utilities](labs/lab-1-xv6-and-unix-utilities.md)
* [Lab 2 Shell](labs/lab-2-shell.md)
* [Lab 3 Buddy Allocator](labs/lab-3-buddy-allocator.md)
* [Lab 4 Lazy](labs/lab-4-lazy.md)
* [Lab 5 Copy-on-Write Fork for xv6](labs/lab-5-copy-on-write-fork-for-xv6.md)
* [Lab 6 RISC-V assembly](labs/lab-6-risc-v-assembly.md)
* [Lab 6 Uthread: switching between threads](labs/lab-6-uthread-switching-between-threads.md)
* [Lab 6 Alarm](labs/lab-6-alarm.md)
* [Lab 7 Lock](labs/lab-7-lock.md)
* [Lab 8 File System: Large Files](labs/lab-8-file-system-large-files.md)
* [Lab 8 File System: Symbolic links](labs/lab-8-file-system-symbolic-links.md)
* [Lab 9 mmap](labs/untitled.md)
* [Lab 10 Networking Part 1](labs/lab-10-networking-part-1.md)
* [Lab 10 Networking Part 2](labs/lab-10-networking-part-2.md)

## Hardware, Device, Assembly

* [RISC-V assembly](hardware-device-assembly/risc-v-assembly.md)
* [Assembly: Access and Store information in Memory](hardware-device-assembly/assembly-access-and-store-information-in-memory.md)
* [Start xv6 and the first process](hardware-device-assembly/start-xv6-and-the-first-process.md)
* [Why first user process loads another program?](hardware-device-assembly/why-first-user-process-loads-another-program.md)
* [What does kernel.ld do in XV6?](hardware-device-assembly/what-does-kernel.ld-do-in-xv6.md)
* [XV6 Device Driver](hardware-device-assembly/xv6-device-driver.md)

