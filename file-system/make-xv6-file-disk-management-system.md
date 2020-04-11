# Make Xv6 File disk management system

QEMU presents a “legacy” virtio interface and has a virtue disk device. Xv6 implements a driver for this disk device. `virtio_disk.c`

QEMU command `qemu … -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0`

The `fs.img` is actually another xv6 problem built, and ask QEMU to treat it as file system.

### Who builds fs.img?

A program called `mkfs.c` does.

It builds disk layout as: `[ boot block | sb block | log | inode blocks | free bit map | data blocks ]`

The program calls Real OS \(not xv6, not QEMU\) `create` system call to create a file named `fs.img`. It set all data to this file! `mkfs` writes all the disk layout content, including superblock, inode, bitmaps, etc.

This program runs before kernel starts.

When kernel starts, it will contains this virtual file system already \(supported by QEMU\)!

