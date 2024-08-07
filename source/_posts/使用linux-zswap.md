---
title: 使用linux zswap
tags:
  - Linux内核
  - zswap
description: 本文简单介绍Linux zswap的使用方式
abbrlink: a9770475
date: 2021-06-27 18:02:47
---

zswap是Linux内核里压缩swap内存的一个特性，他可以把需要swap到swap设备上内存先压缩
下，直到一定的门限值后再向swap设备写入。这个特性可以优化系统内存被大量使用，系统
有swap的时候的系统性能。本文简单介绍怎么使用。

1. 内核需要打开zswap的配置：CONFIG_ZSWAP。还需要打开zswap可能用到的用于压缩
   内存存储的内存分配器，比如: CONFIG_ZBUD

2. 在内核的启动cmdline里加zswap.enable=1可以自动加载zswap模块，zswap.compressor=xxx
   可以选择压缩算法。如果不配置zswap.compressor, 默认的压缩算法是LZO。

3. 启动系统，dmesg | grep zswap可以看到zswap正常加载，并且选择后端压缩内存分配器
   的打印log。
```
	zswap: loaded using pool lzo/zbud
```

4. zswap的控制参数也可以在/sys/module/zswap/parameters/*里配置。

   compressor 选择压缩算法，enabled使能zswap功能，max_pool_percent配置压缩内存
   池占系统内存的百分比(zswap是先把压缩的内存放在一个zpool里), zpool是后端压缩
   内存分配器，这里是zbud, same_filled_pages_enable使能对内存值相同的情况做优化
   处理。

   zswap的debug信息可以在/sys/kernel/debug/zswap/*里显示。

5. 如果系统没有swap分区，测试之前需要给系统加上swap分区，可以对一个空闲的磁盘分
   区做：
```
   mkswap /dev/sda
   swapon /dev/sda
   用swapon -s可以查看系统中已经有的swap分区。
```

6. 如果要看下zswap是否运行，需要构造系统内存被大量使用的场景。如果在服务器上，
   这样的场景不好构造。可以使用cgroup构造。
```
   cgcreate -g memory:bob
   echo 0x4000000 > /sys/fs/cgroup/memory/bob/memory.limit_in_bytes
```
   创建一个名字是bob的memory cgroup，限定在其中的进程使用内存大小是0x40MB。
```
   cgexec -g memory:bob memhog 128m
```
   运行memhog这个程序，这个程序是一个内存测试程序，其操作的虚拟内存大小是128MB.

   运行上面的命令后，查看/sys/kernel/debug/zswap/stored_pages，可以发现其值变
   为非0。

7. 以上的环境可以建在虚拟机里。可以基于[这里](https://wangzhou.github.io/Guest-and-host-communication-for-QEMU/)的方法设置：

   在host上使用 qemu-img create disk.img 1G 生成一个1G大小的虚拟磁盘。再在qemu
   启动命令行里加上 -hdb path_of_disk/disk.img 把这个磁盘加给虚拟机。
   再在虚拟机里配置以上各个步骤即可。如果需要调试带硬件加速的zswap，可以把硬件
   的VF直通到虚拟机里调试。
