---
layout: post
title:  "From allnoconfig to Devuan on EC2"
date:   2019-04-20 11:04:47 +0000
categories: tutorial
---

What to all those linux CONFIG settings to?
How many of them do we actually need?
What's the smallest we can get away with and have something useful?

We're going to go from a hardly functional bare-bones linux kernel, to one that's running on an EC2 instance with the Devuan user space.
All the bits of config that need to be added to get it to work will be spoken about.
QEMU will be used for development and testing along the way.

Let's start minimal, with an `allnoconfig` which has as many things switched off as it can.
```
make allnoconfig
make clean
time make -j8
...
Setup is 13724 bytes (padded to 13824 bytes).
System is 642 kB
CRC c340aac6
Kernel: arch/x86/boot/bzImage is ready  (#39)
make -j8  78.57s user 8.81s system 652% cpu 13.388 total
```

So it took ~13 seconds to build the smallest kernel on 8 cores. How big is it uncompressed and compressed?

```
➜  linux-next git:(master) ✗ ls -lh vmlinux
-rwxr-xr-x 1 jgowans domain^users 1.7M Apr 12 10:56 vmlinux
➜  linux-next git:(master) ✗ ls -lh arch/x86/boot/bzImage
-rw-r--r-- 1 jgowans domain^users 655K Apr 12 10:56 arch/x86/boot/bzImage
```

## Console Output

Let's run it in QEMU and see what it does.
```
qemu-system-x86_64 -nographic \
  -serial mon:stdio \
  -kernel ./arch/x86/boot/bzImage \
  -append "console=ttyS0,38400"
```

Dead quiet... We need to give the kernel serial conole capabilities.
Add the following to the .config
```
CONFIG_PRINTK=y
CONFIG_TTY=y
CONFIG_SERIAL_CORE=y
CONFIG_SERIAL_CORE_CONSOLE=y
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y
```

Now run this to update the config based on what you're just added. Some of the options we've just added "unlock" others so the kernel config generation will prompt you. Things like input, mouse, keyboard PTS, SERIO, etc. We'll get there later, but let's force it to be simple for now. Say no to everything.

```
make oldconfig
```

To speed we can alternatively add some numerican ones manually and pipe 'n' to the rest.

```
CONFIG_PRINTK=y
CONFIGPRINTK_SAFE_LOG_BUF_SHIFT=13
CONFIG_LOG_BUF_SHIFT=17
CONFIG_TTY=y
CONFIG_SERIAL_CORE=y
CONFIG_SERIAL_CORE_CONSOLE=y
CONFIG_SERIAL_8250=y
CONFIG_SERIAL_8250_CONSOLE=y
CONFIG_SERIAL_8250_NR_UARTS=4
CONFIG_SERIAL_8250_RUNTIME_UARTS=4
```

```
yes 'n' | makeoldconfig
...
make -j8
```

Now run it again:
```
qemu-system-x86_64 -nographic \
  -serial mon:stdio \
  -kernel ./arch/x86/boot/bzImage \
  -append "console=ttyS0"
```

Output! Excellent.
Now for a ramdisk.

## BusyBox Initramfs

Before we get to full BusyBox, let's give a simple "hello world" C program as an init process. Put this in init.c
```c
#include <unistd.h>

int main(int argc, char **argv) {
    char *txt = "hello world\n";
    write(1, txt, 12);

    while(1) {
    }
}
```

Compile it and put it in a ramdisk archive.

```
gcc -Wall -g -static -o init init.c
```

Give the kernel some more intramfs functionality:
```
CONFIG_BLK_DEV_INITRD=y
CONFIG_ZLIB_INFLATE=y
CONFIG_DECOMPRESS_GZIP=y
```

Run QEMU again with the ramdisk. QEMU will put the ramdisk into memory; Linux will unpack it and try to run init.

Add an extra cmd line param to QEMU similar to this:
```
-initrd /some/path/to/initramfs.cpio.gz
```

In dmesg, the kernel tries to run init but can't.
```
Run /init as init process
Failed to execute /init (error -2)
```

This is because init uses a shebang; let the kernel know how to execute scripts.

```
CONFIG_BINFMT_SCRIPT=y
```

Trying again:
```
Run /sbin/init as init process
Starting init: /sbin/init exists but couldn't execute it (error -8)
```

Still not, because it can't execute the busybox binary which is referenced in the shebang.
```
CONFIG_ELFCORE=y
CONFIG_BINFMT_ELF=y
```

Still not! Maybe because we're loading a 64-bit binary into a 32-bit system? Make the kernel be 64 bit.

```
CONFIG_64BIT=y
CONFIG_X86_64=y
```

Now it works!
```
Run /init as init process
hello world
```

Time for full busybox. Compile a static busybox binary and symlink farm compressed into a initramfs cpio archive.
[Here's a nice guide.](https://gist.github.com/hardentoo/69409bd33adafc76e5a6a07cc5841007)
Point QEMU to new busybox and there should now be a usable shell!
```
/ # uname -a
Linux (none) 5.6.0-next-20200411+ #59 Sun Apr 12 13:27:45 IST 2020 x86_64 GNU/Lin
```
Whoop whoop.
But some errors in dmesg from the init script:

```
mount: mounting none on /proc failed: No such device
mount: mounting none on /sys failed: No such device
```

Let's fill out some standard debugging parts of the system.

```
CONFIG_PROC_FS=y
CONFIG_PROC_SYSCTL=y
CONFIG_SYSFS=y
CONFIG_DEVTMPFS=y
```

This seems to rebuild everything. I wonder which config option triggered that...? (TODO: make debug)
Now there's dev and proc and sys.
```
/ # ls /dev/
console  kmsg     random   ttyS0    ttyS2    urandom
full     null     tty      ttyS1    ttyS3    zero
/ # ls /sys/
bus       dev       firmware  kernel
class     devices   fs        module
/ # cat /proc/uptime
19.64 19.19
```

## Networking

Let's get some networking. Any devices?

```
/ # ip link
ip: socket: Function not implemented
```

Enable networking in the kernel. 

```
CONFIG_NET=y
CONFIG_INET=y
```

This brings in the potential for lots of stuff. Saying no to all for now.
```
/ # ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
Excellent! Now let's give the guest an emualted e1000 ethernet NIC.

```
CONFIG_HAVE_PCI=y
CONFIG_PCI=y
CONFIG_NETDEVICES=y
CONFIG_ETHERNET=y
CONFIG_NET_VENDOR_INTEL=y
CONFIG_E1000=y
```

This also opens up lots. PCI devices and ethernet devices; both very popular.
```
qemu-system-x86_64 -nographic \
  -serial mon:stdio \
  -kernel ./arch/x86/boot/bzImage \
  -append "console=ttyS0" \
  -initrd /workplace/jgowans/busybox/initrd/initramfs.cpio.gz \
  -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22
```

It's detected and added to the system.
```
e1000: Intel(R) PRO/1000 Network Driver - version 7.3.21-k8-NAPI
e1000: Copyright (c) 1999-2006 Intel Corporation.
e1000 0000:00:03.0: PCI->APIC IRQ transform: INT A -> IRQ 11
e1000 0000:00:03.0 eth0: (PCI:33MHz:32-bit) 52:54:00:12:34:56
e1000 0000:00:03.0 eth0: Intel(R) PRO/1000 Network Connection
```

```
/ # ip link show dev eth0
2: eth0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
```

Could do with an IP address!
```
/ # ip link set dev eth0 up
/ # e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
/ # udhcpc
udhcpc: started, v1.32.0.git
udhcpc: sending discover
udhcpc: sending select for 10.0.2.15
udhcpc: lease of 10.0.2.15 obtained, lease time 86400
```

This doesn't seem to actually assign the IP, so do it myself:
```
/ # ip addr add dev eth0 10.0.2.15/24
/ # ip addr show dev eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
    link/ether 52:54:00:12:34:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 scope global eth0
       valid_lft forever preferred_lft forever
```
See if networking actually works. QEMU will forward from port 5555 on the host into the guest on port 22. Designed for SSH but we can re-purposed for a traffic test.

Guest:
```
/ # nc -l -p 22 -v -w 30
listening on 0.0.0.0:22 ...
process 23 (nc) attempted a POSIX timer syscall while CONFIG_POSIX_TIMERS is not set
```

Fix that up:

```
CONFIG_POSIX_TIMERS=y
```

Host:
```
nc -v localhost 5555
Connection to localhost 5555 port [tcp/*] succeeded!
hello world
```

```
/ # nc -l -p 22 -v -w 30
listening on 0.0.0.0:22 ...
connect to 10.0.2.15:22 from 10.0.2.2:49110 (10.0.2.2:49110)
hello world
```

Excellent! TCP/IP networking working.

Why I didn't need to enable ACPI for the discovery of the card is a bit beyond me. Doesn't PCI need ACPI? Does does the kernel know that the PCI bus exists and where it is? I need to know more about device discovery.

PROBABLY:

CONFIG_UNIX=y
CONFIG_PACKET=y

Known missing:

UNIX
CONFIG_NETDEVICES
CONFIG_PACKET

Maybe:

ETHTOOL_NETLINK

## Block device

Now for a block device. Let's get a full userspace image.
Devuan:
https://mirror.leaseweb.com/devuan/devuan_ascii/virtual/

The image is an ext4 filesystem so we need to give the kernel ext4 capabilities.
I think QEMU will pass an IDE drive through as SATA for the q35 machine. So let's try to get SATA working.
For now let's mount in busybox initramfs.

ext4 file system.


```
CONFIG_BLOCK=y
CONFIG_ATA=y
CONFIG_EXT4_FS=y
```

```
/ # ls /sys/block
/ # 
```

Damn.

lrwxrwxrwx    1 0       0                0 Apr 12 14:41 driver -> ../../../bus/pci/drivers/ahci



CONFIG_BLK_DEV_SD=y
SCSI
ACPI
ATA_ACPI
CONFIG_PCI_MSI=y
CONFIG_PCI_QUIRKS=y

CONFIG_BLK_DEV
BLK_DEV_SD
SATA_AHCI


CONFIG_ATA_SFF=y
CONFIG_ATA_BMDMA=y



ATA_PIIX
PATA_MPIIX
PATA_ACPI
ATA_GENERIC
CONFIG_BLK_DEV_BSG
CONFIG_SATA_PMP=y



