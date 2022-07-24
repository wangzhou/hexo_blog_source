---
title: 使用qemu虚拟机学习Linux内核
tags:
  - QEMU
  - 软件开发
description: >-
  对于linux kernel的初学者来说，搭建一套内核可以运行的环境是非常重要的。看书的同时
  自己动手编译下内核，加点打印看看程序的执行过程是很有帮助的。本文介绍一个这样的简易 环境用于linux kernel的学习。
abbrlink: c5c4ef41
date: 2021-07-17 10:56:21
categories:
---

当然可以在自己的PC机上装一个linux的发行版，然后写一些linux内核模块，插入到当前
的内核中。但是这样会使自己的PC机不稳定，内核模块有bug的话容易使得整个系统崩溃掉。
还有一点，现在做linux kernel开发的很多使用的不是X86的体系架构，如果做arm下的开发,
现在这样做也是不行的。本文以arm64为例子，介绍一个使用qemu虚拟机学习linux kernel
的环境。

0. PC机环境
    本人的PC机上的操作系统是ubuntu 14.04

1. qemu虚拟机
    1. 下载qemu原码：
        git clone git://git.qemu-project.org/qemu.git
	(没有git的 sudo apt-get install git 安装一个)
    2. 编译arm64体系构架的qemu虚拟机：
        cd qemu/
	./configure -target-list=aarch64-softmmu
	make
        在qemu/aarch64-softmmu下会有qemu-system-aarch64, 这就是我们将要用的虚拟机。

2. linux kernel
    1. 下载最新内核源码：
        git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
    2. 编译内核：
        make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
	make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image
        (需要提前下载好arm64的工具链，把工具链的路径加入PATH。这里CROSS_COMPILE示工具链而定)

3. 根文件系统
    根文件系统可以使用linaro网站上发布的openembedded的文件系统，如果遇到该文件系统里没有的
    工具，可以自己下载工具的源码，交叉编译后加入该文件系统中。
    1. 下载openembedded文件系统：
        https://releases.linaro.org/latest/openembedded/aarch64
	中的vexpress64-openembedded_minimal-armv8-gcc-4.9_20141023-693.img.gz
    2. 解压：
        gunzip vexpress64-openembedded_minimal-armv8-gcc-4.9_20140923-688.img.gz
	mv vexpress64-openembedded_minimal-armv8-gcc-4.9_20140923-688.img fs.img
    3. 添加文件：
        sudo kpartx -a fs.img
	(没有kpartx的 sudo apt-get install kpartx 装一个)
	sudo mount /dev/mapper/loopOp2 /mnt
	这时可以访问文件系统，可以把自己编译的一些用户态程序放到文件系统中。初次尝试
        可以不做这一步骤。

4. 运行整个系统
    ./qemu-system-aarch64 -machine virt -cpu cortex-a57 \
    -kernel ~/linux/arch/arm64/boot/Image \
    -drive if=none,file=/home/wangzhou/openembaded/fs.img,id=fs \
    -device virtio-blk-device,drive=fs \
    -append 'console=ttyAMA0 root=/dev/vda2' \
    -nographic
