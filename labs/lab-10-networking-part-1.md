---
description: Complete the implementation of the E1000 networking driver
---

# Lab 10 Networking Part 1

## Background 

We will be using a virtual network device called the E1000 to handle network communication. To xv6 \(and the driver you write\), the E1000 looks like a real piece of hardware connected to a real Ethernet local area network \(LAN\). But in reality, the E1000 your driver will talk to is an emulation provided by qemu, connected to a LAN that is also emulated by qemu. On this LAN, xv6 \(the “guest”\) has an IP address of 10.0.2.15. The only other \(emulated\) computer on the LAN has IP address 10.0.2.2. qemu arranges that when xv6 uses the E1000 to send a packet to 10.0.2.2, it’s really delivered to the appropriate application on the \(real\) computer on which you’re running qemu \(the “host”\).

Testing code is setup sending packets to:

```c
// 10.0.2.2, which qemu remaps to the external host,
// i.e. the machine you’re running qemu on.
dst = (10 << 24) | (0 << 16) | (2 << 8) | (2 << 0);
```

## Task: Network Device Driver

[Lab: networking](https://pdos.csail.mit.edu/6.828/2019/labs/net.html) In this part of the assignment, you will complete the implementation of the E1000 networking driver. So far, code has been provided to discover and initialize the device, and to handle interrupts, but not to send and receive packets. Browse Intel’s [Software Developer's Manual](https://pdos.csail.mit.edu/6.828/2019/readings/hardware/8254x_GBe_SDM.pdf) for the E1000. This manual covers several closely related Ethernet controllers. QEMU emulates the 82540EM. You should skim over chapter 2 now to get a feel for the device. To write your driver, you’ll need to be familiar with chapters 3 and 14, as well as 4.1 \(though not 4.1’s subsections\). You’ll also need to use chapter 13 as reference. The other chapters mostly cover components of the E1000 that your driver won’t have to interact with. Don’t worry about the details right now; just get a feel for how the document is structured so you can find things later. Keep in mind that the E1000 has many advanced features, but you can ignore most of these. Only a small set of basic features is needed to complete the lab assignment.

### Your Job

Your job is to implement support for sending and receiving packets. You’ll need to fill in the missing section in e1000\_recv\(\) and e1000\_transmit\(\), both in kernel/e1000.c.

Both sending and receiving packets is managed by a queue of descriptors that is shared between xv6 and the E1000 in memory. These queues provide pointers to memory locations for the E1000 to DMA \(i.e. transfer\) packet data. They are implemented as circular arrays, meaning that when the card or the driver reach the end of the array, it wraps back around to the beginning. A common abbreviation is to refer to the receive data structures as RX and the transmit data structures as TX.

The E1000 generates an interrupt whenever new packets are received. Your receive code must scan the RX queue to handle each packet that has arrived and deliver its mbuf to the protocol layer by calling net\_rx\(\). struct rx\_desc describes the descriptor format. You will then need to allocate a new mbuf and program it into the descriptor so that the E1000 knows where to place the next payload when it eventually reaches the same location in the array at a later time.

Packet sends are requested by the protocol layer when it calls e1000\_transmit\(\). Your transmit code must enqueue the mbuf into the TX queue. This includes extracting the payload’s location in memory and its length, and encoding this information into a descriptor in the TX queue. struct tx\_desc describes the descriptor format. You will need to ensure that mbufs are eventually freed, but only after the transmission has finished \(the NIC can encode a notification bit in the descriptor to indicate this\).

In addition to reading and writing to the circular arrays of descriptors, you’ll need to interact with the E1000 through memory mapped I/O to detect when new descriptors are available on the receive path and to inform the E1000 that new descriptors have been provided on the transmit path. A pointer to the device’s I/O is stored in regs, and it can be accessed as an array of control registers. You’ll need to use indices E1000\_RDT and E1000\_TDT in particular.

## Solution

### How to discover and initialize the hardware device?

#### DMA for E1000

When creating a direct-map page table for the kernel, map this address for E1000.

```text
// pci.c maps the e1000's registers here.
kvmmap(0x40000000L, 0x40000000L, 0x20000, PTE_R | PTE_W);
```

In PCI-Express initialization, all E1000 registers start from this base address.

```c
// we’ll place the e1000 registers at this address.
// vm.c maps this range.
uint64 e1000_regs = 0x40000000L;
```

#### What is PCI Express

[PCI Express - Wikipedia](https://en.wikipedia.org/wiki/PCI_Express) It is the common [motherboard](https://en.wikipedia.org/wiki/Motherboard) interface for personal computers’ [graphics cards](https://en.wikipedia.org/wiki/Video_card) , [hard drives](https://en.wikipedia.org/wiki/Hard_disk_drive) , [SSDs](https://en.wikipedia.org/wiki/Solid-state_drive) , [Wi-Fi](https://en.wikipedia.org/wiki/Wi-Fi) and [Ethernet](https://en.wikipedia.org/wiki/Ethernet) hardware connections.

#### E1000 hardware definitions

All registers are in [E1000](https://pdos.csail.mit.edu/6.828/2019/readings/hardware/8254x_GBe_SDM.pdf).

```c
#define E1000_RDT      (0x02818/4)  /* RX Descriptor Tail - RW */
#define E1000_RDLEN    (0x02808/4)  /* RX Descriptor Length - RW */
#define E1000_RSRPD    (0x02C00/4)  /* RX Small Packet Detect Interrupt */
#define E1000_TDBAL    (0x03800/4)  /* TX Descriptor Base Address Low - RW */
#define E1000_TDLEN    (0x03808/4)  /* TX Descriptor Length - RW */
#define E1000_TDH      (0x03810/4)  /* TX Descriptor Head - RW */
#define E1000_TDT      (0x03818/4)  /* TX Descripotr Tail - RW */
…
```

#### DMA ring format

```c
/* Transmit Descriptor command definitions [E1000 3.3.3.1] */
#define E1000_TXD_CMD_EOP    0x01 /* End of Packet */
#define E1000_TXD_CMD_RS     0x08 /* Report Status */

/* Transmit Descriptor status definitions [E1000 3.3.3.2] */
#define E1000_TXD_STAT_DD    0x00000001 /* Descriptor Done */

// [E1000 3.3.3]
struct tx_desc
{
  uint64 addr;
  uint16 length;
  uint8 cso;
  uint8 cmd;
  uint8 status;
  uint8 css;
  uint16 special;
};

/* Receive Descriptor bit definitions [E1000 3.2.3.1] */
#define E1000_RXD_STAT_DD       0x01    /* Descriptor Done */
#define E1000_RXD_STAT_EOP      0x02    /* End of Packet */

// [E1000 3.2.3]
struct rx_desc
{
  uint64 addr;       /* Address of the descriptor’s data buffer */
  uint16 length;     /* Length of data DMAed into data buffer */
  uint16 csum;       /* Packet checksum */
  uint8 status;      /* Descriptor status */
  uint8 errors;      /* Descriptor Errors */
  uint16 special;
};
```

#### PCI init

In `pci.c` Tell the e1000 to reveal its registers at physical address 0x40000000.

```text
base[4+0] = e1000_regs;
e1000_init((uint32*)e1000_regs);
```

#### E1000 initialization

`e1000_init(uint32 *xregs)` is called by `pci_init`. `xregs` is the memory address at which the e1000’s registers are mapped. The flows are: 1. Reset the device 2. \[E1000 14.5\] Transmit initialization 3. \[E1000 14.4\] Receive initialization 4. transmitter, receiver control bits 5. Ask E1000 for receive interrupts. So E1000 sends interrupts for every packet.

### How Transmit work in hardware?

![](../.gitbook/assets/image%20%2814%29.png)

`[Head, Tail)` contains all the packets that are about to sent to outside by the hardware \(E1000\).

`Tail` points to the 1st already sent but not yet recycled by software \(OS driver\) buffers.

### Transmit Implementation in Driver

Our logic is when our driver about to send a new package, we: 1. Get current Tail position from hardware register. 2. Get the first not recycled buffer starting from Tail position. 3. Free the old buffer. 4. Save the new packet buffer in this location. Update the length, and other metadata. 5. Tell hardware there is a new ready-to-send packet, by move down the tail position.

```c
int e1000_transmit(struct mbuf **m*)
{
  // the mbuf contains an ethernet frame; program it into
  // the TX descriptor ring so that the e1000 sends it. Stash
  // a pointer so that it can be freed after sending.
  acquire(&e1000_lock);
  // For transmitting, first get the current ring position, using E1000_TDT.
  int pos = regs[E1000_TDT];

  // Then check if the the ring is overflowing. If E1000_TXD_STAT_DD is
  // not set in the current descriptor, a previous transmission is
  // still in flight, so return an error.
  if ((tx_ring[pos].status & E1000_TXD_STAT_DD) == 0) {
    release(&e1000_lock);
    return - 1;
  }

  // Otherwise, use mbuffree() to free the last mbuf that was transmitted
  // with the current descriptor (if there was one).
  struct mbuf *b = tx_mbufs[pos];
  if (b)
    mbuffree(b);

  // Then fill in the descriptor, providing the new mbuf’s head pointer
  // and length.
  tx_ring[pos].addr = (uint64) m->head;
  tx_ring[pos].length = (uint64) m->len;

  // Set the necessary cmd flags (read the E1000 manual)
  // and stash away a pointer to the new mbuf for later freeing.
  tx_ring[pos].cmd |= E1000_TXD_CMD_EOP;
  tx_ring[pos].cmd |= E1000_TXD_CMD_RS;
  tx_mbufs[pos] = m;

  // Finally, update the ring position by adding one to E1000_TDT
  // modulo TX_RING_SIZE
  regs[E1000_TDT] = (pos + 1) % TX_RING_SIZE;
  release(&e1000_lock);
  return 0;
}
```

### How Receiving work in hardware?

Hardware maintains a circular ring of descriptors. Between Head and Tail, are the empty buffers that hardware writes when new packets arrive. As packets arrive, they are stored in memory and the head pointer is incremented by hardware.

![](../.gitbook/assets/image%20%282%29.png)

The `tail` points to the last empty buffer that hardware owns for receiving new packets.

The `tail + 1` points to the 1st fully received buffer that software \(OS driver\) can consume.

After consumption, the tail is moved down by 1, since OS driver wants hardware to know that there is a new empty buffer to use for receiving. 

{% hint style="info" %}
WARNING

The image `tail` is WRONG. It should be 1 entry above. \(See code below\)
{% endhint %}

### Receiving Implementation in Driver

1. Get the received packet at `tail + 1` position.
2. Ensure it is ready to be consumed.
3. Update buffer length from description length.
4. _**Deliver this buffer to protocal layer.**_
5. Allocate a new buffer and set at current position.
6. Since we now have 1 more empty buffer for receiving, tell hardware about this.

   ```c
   static void
   e1000_recv(void)
   {
   // Check for packets that have arrived from the e1000
   // Create and deliver an mbuf for each packet (using net_rx()).
   //
   // First get the next ring position, 
   // using E1000_RDT plus one modulo RX_RING_SIZE.
   acquire(&e1000_lock);
   int cur = (regs[E1000_RDT]+1) % RX_RING_SIZE;

   // Then check if a new packet is available by checking 
   // for the E1000_RXD_STAT_DD bit in the status portion 
   // of the descriptor. If not, stop.
   while ((rx_ring[cur].status & E1000_RXD_STAT_DD) != 0) {
    /*
    Otherwise, update the mbuf's length to the length reported 
    in the descriptor (e.g., use mbufput()). 
    Deliver the mbuf to the protocol layer using net_rx(). 
    (e1000_init() allocates an mbuf for each slot in the receive ring
    initially.)
    */
    int len = rx_ring[cur].length;
    //printf(“I got a packet! Pos(%d). Len(%d)\n”, cur, len);

    mbufput(rx_mbufs[cur], len);

    release(&e1000_lock);
    net_rx(rx_mbufs[cur]);
    acquire(&e1000_lock);

    /*
    Then allocate a new mbuf (because net_rx() maybe hanging on to the mbuf passed to it) and program its head pointer into the descriptor. Clear the descriptor’s status bits to zero.
    */
    rx_mbufs[cur] = mbufalloc(0);
    rx_ring[cur].addr = (uint64) rx_mbufs[cur]->head;
    rx_ring[cur].status = 0; // clear bits

    // Finally, update the E1000_RDT register to the next position 
    // by writing to it.
    regs[E1000_RDT] = cur;
    cur = (regs[E1000_RDT]+1) % RX_RING_SIZE;
   }
   release(&e1000_lock);
   }
   ```

