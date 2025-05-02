+++
title = 'SD Cards'
date = 2025-04-18T18:26:52-05:00
draft = false
author =  'Joseph Wilson'
+++

## Why have an SD Card at all

When we started with TAOS we were trying to run on x86 laptops. 
The one we had focused on was a laptop that had eMMC storage, so 
since eMMC is basically a fancy SD card, The first thing I worked on was 
an SD card driver that could read and write blocks to unblock the file 
system people. 

## Useful reading

The SD association provides [simplified specifications](https://www.sdcard.org/downloads/pls/), 
they might not be enough to use for a commercial grade operating system, but 
they are good enough to use for hobby operating systems. The most important 
documents on this page will be the [SD Host Controller Simplified Specification](https://www.sdcard.org/downloads/pls/pdf/?p=PartA2_SD%20Host_Controller_Simplified_Specification_Ver4.20.jpg&f=PartA2_SD%20Host_Controller_Simplified_Specification_Ver4.20.pdf&e=EN_SSA2)
and the [Physical Layer Simplified Specification](https://www.sdcard.org/downloads/pls/pdf/?p=Part1_Physical_Layer_Simplified_Specification_Ver9.10.jpg&f=Part1PhysicalLayerSimplifiedSpecificationVer9.10Fin_20231201.pdf&e=EN_SS9_1). 
Do not be scared of the large page count, use control F and the table of contents
wisely and you will be fine. 


## Setting it up in QEMU

The first thing you are going to want to do is tell QEMU to emulate an 
SD card. This can be achieved by adding a line like the following
to the command line used to start QEMU.

```sh
    -drive id=mysdcard,file=storage_test.img,if=none,format=raw \
    -device sdhci-pci \
    -device sd-card,drive=mysdcard \
```

In this example `storage_test.img` is the disk image that you
will be using for the device. It can be created in a means like this. 

```sh
	dd if=/dev/zero of=storage_test.img bs=1M count=4k
```

One might wonder why this example produces a 4GiB file of nothing, the reason 
for the size is that there are differences in how drivers work with SDSC cards, 
and SDHC cards, which are SD cards that are bigger than 4GiB. Since at this 
time our group was still hoping to make it onto real hardware I decided to 
only focus on SDHC cards, even if it meant that running the make blank-drive
command created a 4GiB file of nothing.

Now that we have QEMU giving us an sd card, we can look into the QEMU monitor
to verify that the block device was created and works. If you are running with
the nographic option (or for whatever reason the monitor is on stdout / err)
you can use Ctrl-a followed by c to open the QEMU monitor, from there you can 
type info block to see if the SD card made it into the system. If you have a 
graphical QEMU monitor you can use Ctrl-Alt-2 (hold all of them down at once)
and can still use info block to list all of the block devices installed on your
VM.

## PCI Basics

You might have noticed the sdhci-pci in the device. So to use the SD card you 
need to use PCI (Or PCI-Express, but PCI is simpler). The 
[OSDEV article](https://wiki.osdev.org/PCI) on PCI is quite good and explains
how device discovery can work using PCI. If we look Section C.3 of the Host
Controller Simplified Specification we can tell that we 
are looking for a device with a class code of 0x8, and a subclas of 0x5. You 
should also make sure that the programming interface is 0x1 or 0x0. 0x1 implies
that DMA can be used, while 0x0 means DMA can not be used. It being 0x2 means 
that the device has some vendor specific properties, so you should contact
said vendor to determine what said properties are.

So now that we have discovered our SD Card, we need to determine what memory
address to use to communicate with our SD card. Recall that when dealing with 
our PCI bus we used Port Mapped IO, but our SD card expects Memory Mapped IO,
to enable MMIO we need to enable busmastering on the pci bus. But
we need to grab the base address from the Base Address Registers. The QEMU
setup provided will only use the first base address register so 
we can simply grab offset 0x10 from the PCI bus. Do note, the Base Address
Registers are read only. 

Its also important to note that all addresses in Appendix C of the Host 
Controller Specification are port addresses relative to the PCI device's 
address on the PCI bus, again the OSDEV article on PCI is great.


So we have the Base Address Register, what now? First you might want to 
sanity check that it does not point to "real" physical memory. But 
we should not just add this page to our page tables. This is because of our friend
the cache. Caches work great when you are dealing with normal memory, where 
the CPU is the only one who will read or write to it, but this isn't a DRAM
chip that would respond to data accesses, instead this is a device that might do
weird things like return 2 different values for reads to an address when no 
writes have happened. So this means that we need to turn the cache off when 
we are dealing with the SD card. The nicest way to do this is by the Page Attribute Table in X86.
We can verify that our cpu supports this feature by executing 
the CPUID instruction with 1 in EAX and ensuring that bit 16 of EDX is high.

After we know that the PAT is enabled, we want to encode the page of the SD Card
as Strong Uncacheable (UC). If the values of the PAT have remained unchanged
that means we want the PCD and PWT flags to be set in our page table entry 
(Bits 3 and 4 of the Page Tables respectively). 

However, even if our processor no longer caches our stuff and will always go out 
to "memory", Our compiler might mess us up, thinking that 2 reads to the same 
address will get the same result, and will just store the value in a register, 
and not have the second read. So we need to use the idea of volatile, which 
tells the compiler that "hey this isn't memory, if I say to read or write 
something, do it, don't skip it because you think its not needed."

## SD card basics

So now we have our base address register set up and we can access the memory 
of the SD card. How do we actually set it up so we can read and write data
to and from the SD card?

### Software Reset

A software reset is probably the simplest thing needed 
to do. Simply look at the software reset register (located at offset 0x2F 
from the SD card's memory base address register), and write a 1 to it. Then
wait for the value read from this address to return a 0. Now that this is 
done were just going to say all addresses are offsets from the memory
base address register, because im tired of typing that

### Enable Interrupts

Even though this tutorial uses polling we still need to set the interrupt 
status. For normal interrupts the data is at 0x34, while for error 
interrupts it is located at 0x38. You can probably enable all non-reserved 
bits, but I decided to steal free-bsd's masks for these interrupts
which were 0x1FF for normal interrupts, and 0x0FB for error interrupts

### Give it power

Now, even though we are only running in QEMU, QEMU is pretending to be an 
SD Card, so we need to negotiate how much power to give the SD Card.
So step 1 will be to read the SD cards capabilities (at offset 0x40) and 
then see what voltage is supported. Then you will find the power control
register (at offset 0x29) and write 0 to it, turning off power, and then 
you can write the bit pattern representing the maximum voltage supported 
in the capabilities register. After that write, you can or that value with 1
and then write it again (see why we needed volatile writes)

### Set the clock

Step 1 would be to disable the clock, by taking the value in the clock 
control register (offset 0x2c), and unsetting the last bit. We then need
to calculate our clock divisor. Keeping in mind that a normal speed sd card
should have a clock frequency les than 25Mhz. The base clock is in the 
capabilities register. After we set our divisor appropriately, we can 
re-enable the internal clock, and wait to see that the internal clock is 
stable. (Keep in mind that QEMU does not support PLL Enable if you do not 
opt into host controller 4, and this tutorial does not cover host 
controller 4),  we then write to the clock control register

### Enable time outs

You can just max out the timeout parameter (register is 0x2e)

### Actually Start it up

Now we follow the great and lovely flowcharts, They are great, so follow 
Section 3.6. To note when you send commands the response of the command 
is in register 0x10, and it may be  either a 32 bit response, or a 128
bit register. When sending a command you also need to know its response 
type, which we get from the physical layer specification. This alo 
determines flags to send to the sd card.

### Get Data

Now that everything is finished we can set block size and block count, 
then, we will write the sector to the argument register, and set the 
transfer mode. Then we can send the sd command to read / write blocks
and wait for buffer read / write ready interrupt to be generated, and we 
can acknowledge these by writing a 1 to that register position. then since 
we are not using DMA we can just read the data from the buffer data port 
register

## Getting a Filesystem

Once you have the Block Device you need to communicate with who ever is 
working on the filesystem. You should keep in mind that copying data is bad
and try and reduce copies of data as much as possible. You might also want
an abstraction layer between sector numbers, and block numbers, and 
partitions

## Making it fast(er)

Now if something is not as fast as you hoped, one thing that can work is
getting multiple blocks at a time. In my testing, getting 8 blocks at a 
time doubled my speed compared to getting only one block at a time.  