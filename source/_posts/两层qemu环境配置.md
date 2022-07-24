---
title: 两层qemu环境配置
tags:
  - QEMU
  - 虚拟化
description: >-
  本文介绍qemu启动的系统里再起一个qemu的环境配置。这样可以搭建一个完全虚拟的
  虚拟化环境，方便调试虚拟化相关的东西。本文说明整个搭建的逻辑，并搭建一个arm64 的qemu in qemu的环境。
abbrlink: f921b1e9
date: 2021-09-17 22:55:24
categories:
---

第一层qemu
----------

第一层qemu正常编译即可。

第一层qemu启动的时候要打开vcpu的hypvisor extention特性，这样这一层qemu启动的系统
里就会带/dev/kvm，第二层qemu使用kvm加速，第二层qemu的运行速度会比较快，否则第二层
qemu慢的一塌糊涂。

arm构架下，打开vcpu hypvisor extention特性的配置是：
```
	qemu-system-aarch64 -cpu cortex-a57 \
	-smp 1 -m 1024M \
	-nographic \
	-M virt,virtualization=true \
	-kernel ~/Image \
	-append "console=ttyAMA0 earlycon root=/dev/ram rdinit=/init" \
	-initrd ~/rootfs.cpio.gz
```
其中 -M virt,virtualization=true 打开vcpu hypvisor extention。

第二层qemu
----------

第二层qemu在编译的时候要打开kvm的支持：
```
configure --target-list=aarch64-softmmu --enable-kvm
```
理论上同样编译一个arm版本的qemu就好，本地编译一个arm版本的qemu目前已经比较方便，
在一台ubuntu 20.04 arm物理机器上直接编译就好，过程中缺什么库，直接安装就好。

但是，如果你的环境是在一台x86机器上，就需要交叉编译qemu，qemu的所有依赖库都要先
交叉编译下。如果是这种情况，一种比较好的解决办法是编译buildroot，同时指定自己的
qemu代码仓库给buildroot，指定自己的qemu代码仓库是为了随后比较方便修改qemu代码，
重复编译。直接用buildroot自己的qemu仓库的配置，会把qemu代码下载到buildroot/output/build/qemu
下面，去这里修改qemu的代码也是可以的，但是make clean会把output/build下的东西都删掉，
还是指定自己的qemu仓库方便一些。

使用这里介绍的方式，local.mk的配置是：
```
QEMU_OVERRIDE_SRCDIR = <your_local_qemu_path>/qemu/
QEMU_OVERRIDE_SRCDIR_RSYNC_EXCLUSIONS = --include .git
```
buildbood qemu的配置buildroot/package/qemu/qem.mk需要修改：
```
diff --git a/package/qemu/qemu.mk b/package/qemu/qemu.mk
index 88516678d1..a480673a68 100644
--- a/package/qemu/qemu.mk
+++ b/package/qemu/qemu.mk
@@ -221,7 +221,6 @@ define QEMU_CONFIGURE_CMDS
 			--disable-opengl \
 			--disable-vhost-user-blk-server \
 			--disable-virtiofsd \
-			--disable-tests \
 			$(QEMU_OPTS)
 endef
```

buildroot配置的时候，最好使用buildroot toolchain和musl c库，使用uclibc会报找不见
fenv.h的错误。编译qemu的时候只编译package qemu就好，BR2_PACKAGE_QEMU，
BR2_PACKAGE_QEMU_CUSTOM_TARGETS=aarch64-softmmu，BR2_PACKAGE_QEMU_FDT要配置下。

启动第二层qemu的时候加上-enable-kvm的启动参数:
```
	qemu-system-aarch64 \
	-cpu host \
	-m 256M \
	-enable-kvm \
	-nographic \
	-machine virt \
	-kernel ~/Image \
	-append "console=ttyAMA0 earlycon root=/dev/ram rdinit=/init" \
	-initrd ~/rootfs.cpio.gz
```

如上整个环境的逻辑如下图所示：
```
    +---------------+
    |               | <--- arm构架qemu
    |  第二层qemu    |
    |               |
    +---------------+ <--- kvm加速
    +---------------+
    |               | <--- arm构架qemu
    |  第一层qemu    |
    |               |
    +---------------+
    +----------------------------------------------+
    |                                              |
    |  物理机器(可能是arm构架,也可能是x86机器)          |
    |                                              |
    +----------------------------------------------+
```
