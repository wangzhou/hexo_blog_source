---
title: Build your own rootfs by buildroot
date: 2021-06-20 23:16:36
tags: [软件调试, rootfs]
description: "本文简单记录下编译rootfs的过程"
---

1. git clone git://git.buildroot.net/buildroot

2. cd buildroot

3. make defconfig

4. make menuconfig to choose special tools, e.g. numactl

5. make menuconfig to choose BR2_TARGET_ROOTFS_CPIO=y and
   Compression method to gzip.

6. make menuconfig to choose your arch, e.g. my arch is BR2_aarch64
   Note: if you choose a wrong arch, after changing to right arch config,
         you should make clean, than make, otherwise you may meet something
	 wrong like: "No working init found".

7. wait and you will find built minirootfs in buildroot/output/images/
   you can use rootfs.cpio.gz as rootfs here.
