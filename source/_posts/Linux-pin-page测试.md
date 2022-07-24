---
title: Linux pin page测试
tags:
  - Linux内核
description: '本文是对于Linux pin物理页面行为的一个测试, 通过测试确定相关的物理页面确实没有发生移动'
abbrlink: a0f66ae6
date: 2021-06-20 23:14:47
---

我们写一个pin page的代码观察pin page系统的具体行为。这个测试代码分为两部分，一部分
是一个内核驱动，一部分是一个用户态的测试代码。

内核驱动的代码在：https://github.com/wangzhou/tests/tree/master/page_pin
page_pin_test.c, 用户态代码在相同目录的u_page_pin.c里。

内核驱动暴露一个字符设备到用户态，用户态测试代码可以通过这个字符设备的ioctl接口
做pin page的操作。这个操作把用户态申请的VA对应的物理地址pin住，使得对应的物理
地址常驻内存，而且不会在不同numa节点之间迁移。

对应的用户态测试代码先用mmap申请一段匿名页，然后调用上述字符设备的ioctl接口把这
段匿名页pin住。

pin page的过程会触发缺页流程然后拿到具体的物理页。可以在pin ioctl的前后分别加上
getrusage()统计系统缺页的次数, 我们用struct rusage里的ru_minflt统计建立物理页时
的缺页流程。使用如上目录中的u_uacce_pin_page.c可以看到pin ioctl前后的的ru_minflt
差值正好就是mmap申请内存页的数目。

我们使用虚拟机观察内存迁移的情况。具体的虚拟机启动参数如下：
```
./qemu/aarch64-softmmu/qemu-system-aarch64 -machine virt,gic-version=3,iommu=smmuv3 \
-bios ./QEMU_EFI.fd \
-enable-kvm -cpu host -m 4G \
-smp 8 \
-numa node,nodeid=0,mem=2G,cpus=0-3 \
-numa node,nodeid=1,mem=2G,cpus=4-7 \
-kernel ./linux-kernel-warpdrive/arch/arm64/boot/Image \
-initrd ~/repos/buildroot/output/images/rootfs.cpio.gz -nographic -append \
"rdinit=init console=ttyAMA0 earlycon=pl011,0x9000000 acpi=force" \
-device virtio-9p-pci,fsdev=p9fs,mount_tag=p9 \
-fsdev local,id=p9fs,path=p9root,security_model=mapped
```
如上，我们给虚拟机配置了两个numa节点，一个numa节点配置了2G内存。在u_uacce_pin_page.c
里再启动一个线程，并把第二个线程绑定到cpu4，就是numa node1上跑。主线程先分配内存
然后写内存，过10s后，第二个线程写同样的地址。分别测试pin page和没有pin page的情况，
并用numastat的跟踪查看：(下面a.out是u_uacce_pin_page.c编译出来的可执行程序)
```
#!/bin/sh

./a.out &
PID=`pidof a.out`

for i in `seq 10`
do
	numastat -p $PID
	sleep 2
done

kill -9 $PID
```
可以看出，在有pin page的时候，内存始终在一个numa节点上。在没有pin page的时候，
内存可能是一开始在numa node0上，但是10s后被迁移到numa node1上。这个是因为在没有
pin page时，内核会根据实际使用内存的cpu，动态的调整实际物理内存的分布。
```
Per-node process memory usage (in MBs) for PID 417 (a.out)
                           Node 0          Node 1           Total
                  --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.01            0.00            0.01
Stack                        0.00            0.00            0.01
Private                      0.57            0.00            0.57
----------------  --------------- --------------- ---------------
Total                        0.58            0.00            0.59

Per-node process memory usage (in MBs) for PID 417 (a.out)
                           Node 0          Node 1           Total
                  --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.01            0.00            0.01
Stack                        0.00            0.00            0.01
Private                      0.57            0.00            0.57
----------------  --------------- --------------- ---------------
Total                        0.58            0.01            0.59

[...]

Per-node process memory usage (in MBs) for PID 417 (a.out)
                           Node 0          Node 1           Total
                  --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.01            0.00            0.01
Stack                        0.00            0.00            0.01
Private                      0.17            0.40            0.57
----------------  --------------- --------------- ---------------
Total                        0.18            0.41            0.59

Per-node process memory usage (in MBs) for PID 417 (a.out)
                           Node 0          Node 1           Total
                  --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.01            0.00            0.01
Stack                        0.00            0.00            0.01
Private                      0.17            0.40            0.57
----------------  --------------- --------------- ---------------
Total                        0.18            0.41            0.59

[...]
```
