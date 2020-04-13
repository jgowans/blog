---
layout: post
title:  "From allnoconfig kernel to Debian on EC2"
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
CONFIG_PRINTK_SAFE_LOG_BUF_SHIFT=13
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
yes 'n' | make oldconfig
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
CONFIG_RD_GZIP=y
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
CONFIG_PACKET=y
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

## Block device

Now for a block device. Let's get a full userspace image.
Devuan:
[https://mirror.leaseweb.com/devuan/devuan_ascii/virtual/](https://mirror.leaseweb.com/devuan/devuan_ascii/virtual/)

The image is an ext4 filesystem so we need to give the kernel ext4 capabilities.
I think QEMU will pass an IDE drive through as SATA for the q35 machine. So let's try to get SATA working.
For now let's mount in busybox initramfs.

```
(qemu) info qtree
bus: main-system-bus
  type System
  dev: q35-pcihost, id ""
    MCFG = 2952790016 (0xb0000000)
    pci-hole64-size = 34359738368 (32 GiB)
    short_root_bus = 0 (0x0)
    below-4g-mem-size = 134217728 (128 MiB)
    bus: pcie.0
      type PCIE
      dev: e1000, id ""
        mac = "52:54:00:12:34:56"
        netdev = "net0"
      dev: ich9-ahci, id ""
        addr = 1f.2
        class SATA controller, addr 00:1f.2, pci id 8086:2922 (sub 1af4:1100)
        bar 4: i/o at 0xc080 [0xc09f]
        bar 5: mem at 0xfebf1000 [0xfebf1fff
        bus: ide.5
          type IDE
        bus: ide.4
          type IDE
        bus: ide.3
          type IDE
        bus: ide.2
          type IDE
        bus: ide.2
          type IDE
          dev: ide-cd, id ""
            drive = "ide2-cd0"
        bus: ide.0
          type IDE
          dev: ide-hd, id ""
            drive = "jgowansvol"       <------------ HERE
            logical_block_size = 512 (0x200)
            physical_block_size = 512 (0x200)
```

So... TODO.

Exactly how AHCI, IDE, IDE-HD, SATA, PATA, SCSI and PCI all tie in together I'm a bit unsure of. For a start I find it strange that IDE and AHCI appear together, strange that it's a SATA controller with an IDE drive, and strange that when it's being detected on boot by Linux it mentioned SCSI

TODO split pre post first enum

Anyway, my guess from this device tree (I don't know how to do better than guess?) is that I'll need AHCI drivers, SATA drivers, and maybe even SCSI drivers.

```
CONFIG_BLOCK=y
CONFIG_ATA=y
CONFIG_SATA_AHCI=y
```

Now the QEMU HDD is picked up:
```
ata1: SATA link up 1.5 Gbps (SStatus 113 SControl 300)
ata1.00: ATA-7: QEMU HARDDISK, 2.5+, max UDMA/100
ata1.00: 41940992 sectors, multi 16: LBA48 NCQ (depth 32)
ata1.00: applying bridge limits
ata1.00: configured for UDMA/100
scsi 0:0:0:0: Direct-Access     ATA      QEMU HARDDISK    2.5+ PQ: 0 ANSI: 5
```

But it's not enumerated as a block device.
```
/ # ls /sys/block
/ # 
```

Somehow (I think comparing against a full running system?) I found adding this guy does the job:
```
CONFIG_BLK_DEV_SD=y
```

```
sd 0:0:0:0: [sda] 41940992 512-byte logical blocks: (21.5 GB/20.0 GiB)
sd 0:0:0:0: [sda] Write Protect is off
sd 0:0:0:0: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
 sda: sda1
 sd 0:0:0:0: [sda] Attached SCSI disk
```

```
/ # ls /sys/block
sda

/ # ls -l /dev/sda*
brw-------    1 0        0           8,   0 Apr 13 09:26 /dev/sda
brw-------    1 0        0           8,   1 Apr 13 09:26 /dev/sda1

/ # mkdir -p /mnt/devuan
/ # mount /dev/sda1 /mnt/devuan/
mount: mounting /dev/sda1 on /mnt/devuan/ failed: No such file or directory

/ # cat /proc/filesystems
nodev	sysfs
nodev	tmpfs
nodev	bdev
nodev	proc
nodev	devtmpfs
nodev	sockfs
nodev	pipefs
nodev	ramfs
```

Seem to be missing ext4 support.
```
CONFIG_EXT4_FS=y
```

```
/ # mount /dev/sda1 /mnt/devuan/
EXT4-fs (sda1): mounted filesystem with ordered data mode. Opts: (null)
/ # ls /mnt/devuan/
bin             initrd.img.old  opt             sys
boot            lib             proc            tmp
dev             lib64           root            usr
etc             lost+found      run             var
home            media           sbin            vmlinuz
initrd.img      mnt             srv             vmlinuz.old
```

Eggcellent! We're going to get to booting a full userspace in a moment, but let's add some performance before going on.
Add `-enable-kvm` to QEMU. Switch the QEMU ethernet and block device to virtio:

## VirtIO

```
qemu-system-x86_64 -nographic -M q35 -enable-kvm \
  -serial mon:stdio \
  -kernel ./arch/x86/boot/bzImage \
  -initrd /workplace/jgowans/busybox/initrd/initramfs.cpio.gz \
  -device virti-net,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 \
  -append "console=ttyS0" \
  -drive file=$HOME/devuan_ascii_2.0.0_amd64_qemu.qcow2,if=virtio,id=jgowansvol
```

```
(qemu) info qtree
bus: main-system-bus
  type System
  dev: q35-pcihost, id ""
    bus: pcie.0
      type PCIE
      dev: virtio-blk-pci, id ""
        disable-legacy = "off"
        disable-modern = false
        addr = 03.0
        class SCSI controller, addr 00:03.0, pci id 1af4:1001 (sub 1af4:0002)
        bar 0: i/o at 0xc000 [0xc07f]
        bar 1: mem at 0xfebf1000 [0xfebf1fff]
        bar 4: mem at 0xfe000000 [0xfe003fff]
        bus: virtio-bus
          type virtio-pci-bus
          dev: virtio-blk-device, id ""
            drive = "rootvol"
            logical_block_size = 512 (0x200)
            physical_block_size = 512 (0x200)
            scsi = false
      dev: virtio-net-pci, id ""
        addr = 02.0
        class Ethernet controller, addr 00:02.0, pci id 1af4:1000 (sub 1af4:0001)
        bus: virtio-bus
          type virtio-pci-bus
          dev: virtio-net-device, id ""
```

```
CONFIG_VIRTIO_MENU=y
CONFIG_VIRTIO_PCI=y
CONFIG_NET_CORE=y
CONFIG_VIRTIO_NET=y
CONFIG_BLK_DEV=y
CONFIG_VIRTIO_BLK=y
```

```
/ # ls -l /sys/class/net/
total 0
lrwxrwxrwx    1 0        0                0 Apr 13 10:07 eth0 -> ../../devices/pci0000:00/0000:00:02.0/virtio0/net/eth0
lrwxrwxrwx    1 0        0                0 Apr 13 10:07 lo -> ../../devices/virtual/net/lo
/ # ls -l /sys/block
total 0
lrwxrwxrwx    1 0        0                0 Apr 13 10:07 vda -> ../devices/pci0000:00/0000:00:03.0/virtio1/block/vda
```

## Devuan Userspace

We're not going to use an initramfs any more; we "just" build all we need into the kernel.
The Devuan image will be mounted as root and the kernel will execute init in it.

```
qemu-system-x86_64 -nographic -M q35 -enable-kvm \
  -serial mon:stdio \
  -kernel ./arch/x86/boot/bzImage \
  -device virtio-net,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 \
  -append "console=ttyS0 root=/dev/vda1" \
  -drive file=$HOME/devuan_ascii_2.0.0_amd64_qemu.qcow2,if=virtio,id=devuan
```

```
VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
Please append a correct "root=" boot option; here are the available partitions:
fe00        20970496 vda 
 driver: virtio_blk
  fe01        20969472 vda1 d32edf1b-01
```

```
-append "console=ttyS0 root=/dev/vda1"
```

```
Run /sbin/init as init process
clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x2b3e08e4177, max_idle_ns: 440795321802 ns
clocksource: Switched to clocksource tsc
udevd[120]: error getting socket: Address family not supported by protocol
udevd[120]: error initializing udev control socket
random: fast init done
EXT4-fs (vda1): re-mounted. Opts: (null)
```

Then crickets. So something happened, but I guess the init process didn't log much?
Debug time. Idea is to start sh as init, then strace and exec SysV init.

```
-append "console=ttyS0 root=/dev/vda1 init=/bin/sh
```

```
# strace -vf -p 1 -e process &
# /bin/sh: 0: Can't open /dev/null
```

Not sure why you need that, but ok.

```
CONFIG_BLK_DEV_NULL_BLK=y
```

```
mount -t devtmpfs none /dev/
# strace -vf -p 1 -e process &
/bin/sh: 3: strace: not found
```

Gruble grumble! Okay, boot Devuan in full and install stace.

```
qemu-system-x86_64 -enable-kvm \
-device virtio-net,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 \
$HOME/devuan_ascii_2.0.0_amd64_qemu.qcow2
```
Connect via VNC, login with root/toor and apt-get install strace.


```
mount -t devtmpfs none /dev/
# exec /sbin/init
INIT: version 2.88 booting
[info] Using makefile-style concurrent boot in runlevel S.
ERROR: could not open /proc/stat: No such file or directory
Starting fake hwclock: loading system time.
Current system time: 2020-04-13 10:43:29
fake-hwclock saved clock information is in the past: 2020-04-13 10:39:38
To set system time to this saved clock anyway, use "force"
[....] Starting the hotplug events dispatcher: udevdudevd[140]: error getting socket: Address family not supported by protocol
udevd[140]: error initializing udev control socket
```

Wait, so it was just /dev/ missing? Hm? Surely sysvinit automounts /dev/?
No, it's probably the initramfs that does it! Let's see what the initramfs does. Roughly:

```
sudo guestmount -a $HOME/devuan_ascii_2.0.0_amd64_qemu.qcow2 -i --ro /mnt/guest
cp /mnt/guest/initrd.img-4.9.0-6-amd64 /tmp/devuan-initrd/devuan-initrd.img-4.9.0-6-amd64
sudo guestunmount /mnt/guest

➜  /tmp file devuan-initrd.img-4.9.0-6-amd64 
devuan-initrd.img-4.9.0-6-amd64: gzip compressed data, last modified: Wed Jun  6 12:26:40 2018, from Unix
zcat devuan-initrd.img-4.9.0-6-amd64| cpio -idmv
less init
```

Indeed, there's all sorts of directory creating and mounting going on there. It' a comprehensive initrd. As well as mounting it probes modules, looks at cmdline args and config, architecture, repairs filesystems, and more. To complex for this guide. Maybe /dev/ is all that's missing? There's a config for that.

```
CONFIG_DEVTMPFS_MOUNT=y
```

Removing `init=/bin/sh` from cmdline and woohoo there's output from sysvinit! It's very unhappy though.

```
[warn] Filesystem type 'devpts' is not supported. Skipping mount. ... (warning).
[....] Configuring network interfaces...error getting socket: Address family not supported by protocol
ifup: waiting for lock on /run/network/ifstate.lo
ifup: failed to lock lockfile /run/network/ifstate.lo: Permission denied
failed.
[....] Starting enhanced syslogd: rsyslogdThe futex facility returned an unexpected error code.Aborted
 failed!
[....] Starting periodic command scheduler: croncron: can't lock /var/run/crond.pid, otherpid may be 32692: Function not implemented
 failed!
The futex facility returned an unexpected error code.Aborted
 failed!
```

Okay, some networking stuff which is meh for now, but it clearly wants a fuxtex.

```
CONFIG_FUTEX=y
```

```
INIT: Entering runlevel: 2
ifup: failed to lock lockfile /run/network/ifstate.lo: Permission denied
failed.
[....] Starting periodic command scheduler: croncron: can't lock /var/run/crond.pid, otherpid may be 32754: Function not implemented
 failed!
```

Progress, now:

```
CONFIG_FILE_LOCKING=y
```

A bit more progress, but now the console just hangs without obvious errors. Back to debugging by setting `init=/bin/sh` and stracing stuff.

By default it seems devuan doesn't spawn a login session on the console? Get into single user mode to force a console.

```
-append "console=ttyS0 rw root=/dev/vda1 single"
```

`cat /etc/inittab` shows some interesting lines. This is why I'm using Devuan, btw, the SystemV init is more approachable, IMO.

```
# What to do in single-user mode.
~~:S:wait:/sbin/sulogin

# Example how to put a getty on a serial line (for a terminal)
#
# T0:23:respawn:/sbin/getty -L ttyS0 9600 vt100
```

Well, that explains how we get a shell in single mode, and if we just uncomment that line and reboot we get a login prompt in normal mode.

Logged in. Start with networking; it would be great to get sshd to start.

```
dhclient -v eth0
# curl example.com
<!doctype html>
<html>
<head>
    <title>Example Domain</title>

...
```

Cool, now sshd?

```
killall sshd; /usr/sbin/sshd -Dde
``` 

Trying to connect:

```
root@(none):/# killall sshd; /usr/sbin/sshd -Dde
debug1: sshd version OpenSSH_7.4, OpenSSL 1.0.2l  25 May 2017
debug1: private host key #0: ssh-rsa SHA256:z4vJBVgVP2omyKAisN/Du2OYMpAyMgmMof1gg30UPZc
debug1: private host key #1: ecdsa-sha2-nistp256 SHA256:CnKOA3yAWi8niDSY+wPkmdmHFAmc5gyQJHMJiOIeBPs
debug1: private host key #2: ssh-ed25519 SHA256:8JZzFKornbx1gEqPxt/fPePOPbYsvSxe9FttBiWqgAQ
debug1: setgroups() failed: Function not implemented
debug1: rexec_argv[0]='/usr/sbin/sshd'
debug1: rexec_argv[1]='-Dde'
debug1: Set /proc/self/oom_score_adj from 0 to -1000
socket: Address family not supported by protocol
debug1: Bind to port 22 on 0.0.0.0.
Server listening on 0.0.0.0 port 22.
reexec socketpair: Address family not supported by protocol
```

Damn.

```
strace -f /usr/sbin/sshd -Dde
socketpair(AF_UNIX, SOCK_STREAM, 0, 0x7ffc52260c68) = -1 EAFNOSUPPORT (Address family not supported by protocol)
socketpair(AF_UNIX, SOCK_STREAM, 0, 0x7ffc52260c68) = -1 EAFNOSUPPORT (Address family not supported by protocol)
```

Need UNIX sockets.

```
CONFIG_UNIX=y
```

```
getsockname(3, {sa_family=AF_INET, sin_port=htons(22), sin_addr=inet_addr("10.0.2.15")}, [128->16]) = 0
write(2, "Connection from 10.0.2.2 port 41"..., 58Connection from 10.0.2.2 port 41252 on 10.0.2.15 port 22
write(2, "debug1: Client protocol version "..., 94debug1: Client protocol version 2.0; client software version OpenSSH_7.6p1 Ubuntu-4ubuntu0.3
) = 94
write(2, "debug1: match: OpenSSH_7.6p1 Ubu"..., 79debug1: match: OpenSSH_7.6p1 Ubuntu-4ubuntu0.3 pat OpenSSH* compat 0x04000000
) = 79
write(2, "debug1: Local version string SSH"..., 69debug1: Local version string SSH-2.0-OpenSSH_7.4p1 Debian-10+deb9u3
) = 69
write(2, "debug1: Enabling compatibility m"..., 54debug1: Enabling compatibility mode for protocol 2.0
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, child_tidptr=0x7f14b7d9e010) = 55
close(4)                                = 0
close(7)                                = 0
poll([{fd=5, events=POLLIN}, {fd=6, events=POLLIN}], 2, -1strace: Process 55 attached
```

Nice, got the connection, negotiated and forked! But then:

```
[pid    55] setgroups(1, [65534])       = -1 ENOSYS (Function not implemented)
[pid    55] write(7, "\0\0\0+\0\0\0\1\0\0\0#setgroups: Function "..., 47 <unfinished ...>
[pid    54] read(6, "\0\0\0\1\0\0\0#setgroups: Function not "..., 43) = 43
[pid    54] write(2, "setgroups: Function not implemen"..., 47setgroups: Function not implemented [preauth]
```

Enable groups with:

```
CONFIG_MULTIUSER=y
```

SSH works now! It's kind of werid, we don't get a shell prompt, but commands do work.

In sshd debug logs:


```
debug1: Allocating pty.
openpty: No such file or directory
session_pty_req: session 0 alloc failed
```

Give it pseudo terminals and mount them (if they're not automounted?)

```
CONFIG_UNIX98_PTYS=y
```

```
mount -t devpts none /dev/pts
```

Now SSH works properly and we get a pty:

```
u8657fce2533d57 ➜  ~ ssh root@localhost -p 5555
root@localhost's password: 
Linux (none) 5.6.0-next-20200411+ x86_64 GNU/Linux

The programs included with the Devuan GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Devuan GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Apr 13 12:34:40 2020
debug1: PAM: reinitializing credentials
debug1: permanently_set_uid: 0/0
Environment:
  LANG=en_US.UTF-8
  LC_CTYPE=en_US.UTF-8
  USER=root
  LOGNAME=root
  HOME=/root
  PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
  MAIL=/var/mail/root
  SHELL=/bin/bash
  SSH_CLIENT=10.0.2.2 41422 22
  SSH_CONNECTION=10.0.2.2 41422 10.0.2.15 22
  SSH_TTY=/dev/pts/0
  TERM=xterm-256color
root@(none):~# ls
console  kmesg	tty  tty1  ttyS0  ttyS1  xconsole
```

Full monty:

```
exec /sbin/init
...
[ ok ] Starting enhanced syslogd: rsyslogd.
[ ok ] Starting SMP IRQ Balancer: irqbalance.
[ ok ] Starting NTP server: ntpd.
[ ok ] Starting periodic command scheduler: cron.
[ ok ] Starting OpenBSD Secure Shell server: sshd
```

SSH works now, as does apt-get.

```
root@devuan:/# apt-get update
Hit:1 http://pkgmaster.devuan.org/merged ascii InRelease
Hit:2 http://pkgmaster.devuan.org/merged ascii-updates InRelease
Hit:3 http://pkgmaster.devuan.org/merged ascii-security InRelease
Reading package lists... Done
```

It's far from complete, but it is a usable, debugable system. :-)

## Booting Directly

Currently we've giving QEMU the kernel which it's loading.
In a standalone system (objective is to run this as an EC2 instance, remember?) GRUB should load our kernel.

Mount the guest file system and copy the kernel in

```
/mnt/guest/boot# cp /workplace/jgowans/linux-next/arch/x86/boot/bzImage vmlinuz-5.6.0
```

Change `/etc/default/grub`

```
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0"
GRUB_TERMINAL="serial"
GRUB_SERIAL_COMMAND="serial --speed=9600 --unit=0 --word=8 --parity=no --stop=1"
```

Rebuild grub config and it should add the new kernel

```
root@devuan:/boot# grub-mkconfig grub2-mkconfig -o /boot/grub/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.6.0
Found linux image: /boot/vmlinuz-4.9.0-6-amd64
Found initrd image: /boot/initrd.img-4.9.0-6-amd64
done
```

We get grub over console (no more VGA!) and it boots! :-)

## EC2 Guest

```
Devuan Runit 2020-04-09 (Unofficial) - ami-0c14b3ee6967c65f3
```

```
admin@ip-172-31-7-117:~$ uname -a
Linux ip-172-31-7-117 5.6.3 #1 SMP Thu Apr 9 02:00:32 CEST 2020 x86_64 GNU/Linux
admin@ip-172-31-7-117:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Devuan GNU/Linux 3 (beowulf)
Release:    3
Codename:   beowulf
```

EC2 uses NVMe for block IO and ENA for the network device.

```
CONFIG_NVMEM=y
CONFIG_BLK_DEV_NVME=y
CONFIG_NET_VENDOR_AMAZON=y
CONFIG_PCI_MSI=y
CONFIG_ENA_ETHERNET=y
```

```
cp /tmp/bzImage /boot/vmlinuz-5.6.0
rm /boot/vmlinuz-5.6.3 
vim /etc/default/grub
```

```
root@ip-172-31-7-117:/boot# grub-mkconfig -o /boot/grub/grub.cfg 
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-5.6.0
done

```

Stop/start the instance, and:

```
admin@ip-172-31-7-117:~$ uname -a
Linux ip-172-31-7-117 5.6.0-next-20200411+ #107 Mon Apr 13 19:11:15 IST 2020 x86_64 GNU/Linux
```

Woohoo! 

dmesg is littered with:

```
[  391.735435] elogind[1145]: Failed to mount cgroup at /sys/fs/cgroup/elogind: No such file or directory
[  391.737251] elogind[1145]: Failed to allocate manager object: No such file or directory
```

Clean that up:

```
CONFIG_CGROUPS=y
```

More spewings in dmesg tracked down via strace:

```
[pid  1900] epoll_create1(EPOLL_CLOEXEC) = -1 ENOSYS (Function not implemented)
[pid  1900] close(4)                    = 0
[pid  1900] writev(3, [{iov_base="<35>", iov_len=4}, {iov_base="elogind", iov_len=7}, {iov_base="[1900]: ", iov_len=8}, {iov_base="Failed to allocate manager objec"..., iov_len=59}, {iov_base="\n", iov_len=1}], 5) = 79
```

```
[pid  2486] signalfd4(-1, [HUP], 8, SFD_CLOEXEC|SFD_NONBLOCK) = -1 ENOSYS (Function not implemented)
[pid  2486] writev(3, [{iov_base="<35>", iov_len=4}, {iov_base="elogind", iov_len=7}, {iov_base="[2486]: ", iov_len=8}, {iov_base="Failed to register SIGHUP handle"..., iov_len=59}, {iov_base="\n", iov_len=1}], 5) = 79

```

```
[pid  1589] timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC|TFD_NONBLOCK) = -1 ENOSYS (Function not implemented)
```

```
[pid  1589] inotify_init1(IN_NONBLOCK|IN_CLOEXEC) = -1 ENOSYS (Function not implemented)
```

```
CONFIG_EPOLL=y
CONFIG_SIGNALFD=y
CONFIG_TIMERFD=y
CONFIG_INOTIFY_USER=y
```

Happy. :-)

```
admin@ip-172-31-7-117:~$ dmesg | tail
[    1.121661] random: udevd: uninitialized urandom read (16 bytes read)
[    1.122836] random: udevd: uninitialized urandom read (16 bytes read)
[    1.158364] udevd[70]: starting eudev-3.2.7
[    1.230715] EXT4-fs (nvme0n1p1): re-mounted. Opts: nodiscard,errors=remount-ro
[    1.237911] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x24095dc534b, max_idle_ns: 440795234041 ns
[    1.239873] clocksource: Switched to clocksource tsc
[    1.886516] elogind[242]: New seat seat0.
[    3.001944] random: crng init done
[    3.002468] random: 7 urandom warning(s) missed due to ratelimiting
[    7.367772] elogind[242]: New session c1 of user admin.
```

## Conclusions

We've gone from a skeleton kernel to one that runs Devuan (maybe Systemd too?) on an EC2 instance.
Only "essential" functionality has been added, where "essential" is defined somewhere between what's necessary to make it work, and what's necessary to make it stop spitting out warning messages.

It's grown about 8x since we started:

```
➜  linux-next git:(master) ✗ grep "^CONFIG_" .config | wc -l
459
➜  linux-next git:(master) ✗ ls -lh vmlinux
-rwxr-xr-x 1 jgowans domain^users 14M Apr 13 20:00 vmlinux
➜  linux-next git:(master) ✗ ls -lh arch/x86/boot/bzImage 
-rw-r--r-- 1 jgowans domain^users 2.2M Apr 13 20:00 arch/x86/boot/bzImage
```

Full build from clean has doubled. But it's still fast!

```
➜  linux-next git:(master) ✗ time make -j8
...
  BUILD   arch/x86/boot/bzImage
Setup is 13724 bytes (padded to 13824 bytes).
System is 2142 kB
CRC 96cabded
Kernel: arch/x86/boot/bzImage is ready  (#112)
make -j8  205.53s user 20.30s system 735% cpu 30.695 total
```
