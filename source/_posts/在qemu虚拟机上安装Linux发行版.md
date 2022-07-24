---
title: 在qemu虚拟机上安装Linux发行版
tags:
  - QEMU
  - 虚拟化
description: >-
  一般使用qemu调试代码的时候我们习惯使用内存文件系统，使用内存文件系统的缺点 是 1. 它是非持久的，无法保存文件系统里的新数据； 2.
  一般内存文件系统都比较 简单，测试的时候可能缺工具(比如我们想用sysbench测试下mysql的性能)。本文
  介绍的是在qemu虚拟机上安装标准发行版的方法，使用这样的文件系统可以克服上面 提到的两个缺点。本文的测试系统是ubuntu18.04 x86构架CPU
abbrlink: 62176ebc
date: 2021-06-19 09:49:08
---

 1. qemu-img create -f qcow2 debian.img 10G
 
 2. sudo kvm -hda debian.img -cdrom debian-10.2.0-amd64-netinst.iso -m 2048

    这一步是把这个debian的iso安装到debian.img这个文件上。
 
 3. qemu command:
 ```
 qemu-system-x86_64 -cpu host -enable-kvm -smp 4 \
 -m 1G \
 -kernel ~/repos/linux/arch/x86/boot/bzImage \
 -append "console=ttyS0 root=/dev/sda1" \
 -hda ./debian.img \
 -nographic \
 ```
    如上启动虚拟机，可以发现自己已经可以使用如上安装的debian系统。我们在第二步
    安装系统的时候可以把需要的程序都装上，在这样的虚拟机里做测试，会方便很多。
    而且你在虚拟机里创建的文件下次启动虚拟机的时候都还在。
