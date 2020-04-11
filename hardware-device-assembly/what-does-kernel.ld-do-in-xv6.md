# What does kernel.ld do in XV6?

## system/kernel/boot

There is a kernel.ld file in xv6.

```text
OUTPUT_ARCH( "riscv" )
ENTRY( _entry )

SECTIONS
{
  /*
   * ensure that entry.S / _entry is at 0x80000000,
   * where qemu's -kernel jumps.
   */
  . = 0x80000000;
  .text :
  {
    *(.text)
    . = ALIGN(0x1000);
    *(trampsec)
  }

  . = ALIGN(0x1000);
  PROVIDE(etext = .);

  /*
   * make sure end is after data and bss.
   */
  .data : {
    *(.data)
  }
  .bss : {
    *(.bss)
    *(.sbss*)
     PROVIDE(end = .);
  }
}
```

It tells linker to build the executable image entry point to `_entry`. `. = 0x80000000;` ensures the first line of `_entry` starts from 0x80000000. See the generated ASM:

```text
kernel/kernel:     file format elf64-littleriscv


Disassembly of section .text:

0000000080000000 <_entry>:
    80000000:    00003117              auipc    sp,0x3
    80000004:    80010113              addi    sp,sp,-2048 # 80002800 <stack0>
    80000008:    6505                    lui    a0,0x1
```

The will let boot loader to load this kernel image to the specified memory position. Note: `qemu -kernel` starts at 0x1000. And the code in 0x1000 jumps to 0x8000000. That’s why we must define our linker config file so that qemu could find the correct address and code to execute.

If we don’t put any address in linker config file, by default, the image files are generated starting from address 0. But our case is special, it is a kernel image, and `qemu -kernel` knows that. Qemu will jump to 0x8000000 no matter what, so we have to put our code starting from that address.

#### Summary

qemu load the kernel image, and put in memory from the specified address. Then it jumps to 0x8000000 and starts from there!

