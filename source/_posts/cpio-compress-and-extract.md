---
title: cpio compress and extract
tags:
  - shell
description: 本文简单记录cpio.gz格式的压缩和解压缩命令
abbrlink: 98894f7b
date: 2021-07-11 23:37:35
categories:
---

we often see a file is a *.cpio.gz, normally it is a gzip compressed data.

"gunzip file.cpio.gz" to extract it to a file.cpio, whichi is a ASCII cpio archive.
"cpio -ivmd < file.cpio" can extract it finally.

we can use "find . | cpio -o --format=newc > ../file.cpio" to compress a directory
to file.cpio. this command should be run in the root of the directory.

then we use "gzip -c file.cpio > file.cpio.gz" to get orignal gzip compressed file.

Above initrd file system should be cpio.gz type. for example, we can run a qemu
besed on memory initrd fiel system like:

```
qemu-system-aarch64 -machine virt -cpu cortex-a57 \
-m 300M \
-kernel ~/repos/linux/arch/arm64/boot/Image \
-initrd ~/rootfs/rootfs.cpio.gz \
-append "console=ttyAMA0 root=/dev/ram rdinit=/init" \
-nographic \
```
Note: as we run rootfs in memory, -m parametre above should carefully choose.
      too small is a bad idea.
