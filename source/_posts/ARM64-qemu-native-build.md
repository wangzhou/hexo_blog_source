---
title: ARM64 qemu native build
tags:
  - QEMU
  - 虚拟化
description: 本文记录在ARM64平台上编译QEMU的过程
abbrlink: 517dc1f6
date: 2021-07-11 23:40:33
categories:
---

0. cd qemu; mkdir build; cd build;

1. install pkg-config

2. install zlib1g-dev zlib1g

3. ERROR: glib-2.22 gthread-2.0 is required to compile QEMU
   install libglib2.0-dev

4. ERROR: pixman >= 0.21.8 not present. Your options: ...
   install libpixman-1-dev

5. ERROR: DTC (libfdt) version >= 1.4.0 not present. Your options: ...
   install libfdt-dev

6. ./configure --target-list=aarch64-softmmu --enable-fdt --enable-kvm --disable-werror

7. make

8. make install # better uninstall previous qemu version if had one.

9. qemu-system-aarch64 --version

    root@linaro-developer:~/qemu/build# qemu-system-aarch64 --version
    QEMU emulator version 2.5.50, Copyright (c) 2003-2008 Fabrice Bellard

In aarch64 ubuntu-16.04, there is doc: cnblogs.com/from-zero/p/14327440.html
