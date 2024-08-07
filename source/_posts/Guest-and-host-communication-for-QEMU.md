---
title: Guest and host communication for QEMU
tags:
  - QEMU
description: >-
  When we use qemu to help debug, we need a way to move files between host and
  guest. This doc shares a easy way to do this.
abbrlink: ec556da6
date: 2021-06-28 23:58:20
categories:
---

We can use 9p fs to do this, qemu cmdline like  below:
```
qemu-system-aarch64 -machine virt -cpu cortex-a57 \
-m 2G \
-kernel ~/repos/kernel-dev/arch/arm64/boot/Image \
-initrd ~/repos/buildroot/output/images/rootfs.cpio.gz \
-append "console=ttyAMA0 root=/dev/ram rdinit=/init" \
-fsdev local,id=p9fs,path=p9root,security_model=mapped \
-device virtio-9p-pci,fsdev=p9fs,mount_tag=p9 \
-nographic \
```
Here path=p9root is the directory which we can see in host and guest.
We can use path=/home/your_account/p9root for example also, but it should be a
full path.

Then we can start qemu, and in qemu do:
```
mount -t 9p p9 /mnt
```

Then you can see the files in host p9root directory in guest /mnt.

Then we can also debug the kernel running in qemu by gdb. We should add
"-gdb tcp::1234" to start a gdb server in qemu and wait on local port 1234 of tcp.
Whole qemu cmdline is like:
```
qemu-system-aarch64 -machine virt -cpu cortex-a57 \
-m 2G \
-kernel ~/repos/kernel-dev/arch/arm64/boot/Image \
-initrd ~/repos/buildroot/output/images/rootfs.cpio.gz \
-append "console=ttyAMA0 root=/dev/ram rdinit=/init" \
-fsdev local,id=p9fs,path=p9root,security_model=mapped \
-device virtio-9p-pci,fsdev=p9fs,mount_tag=p9 \
-nographic \
-gdb tcp::1234
```
After kernel in qemu boots up, you can start a gdb cline in host and connect to
the gdb server in qemu.
```
aarch64-linux-gnu-gdb
(gdb) file ~/repos/kernel-dev/vmlinux
(gdb) target remote:1234
```
Here we use a arm64 based gdb as an example. After "target remote:1234", we
can use gdb to debug kernel in qemu. You can just set break points, print value
of variables...

Or you can -s -S in qemu command line, then in another window, run gdb;
target remote:1234 to connect gdb server in qemu. Then you can set breakpoint,
run and so on. If you have done like above steps, but fail to set a breakpoint,
please disable kaslr, and retry it. For disable kaslr, adding nokaslr in kernel
command line, you can do this in qemu -append, like, -append "xxxx nokaslr"

For debug qemu itself, you can run qemu and get the pid of qemu, then run gdb;
attach <qemu_pid>
