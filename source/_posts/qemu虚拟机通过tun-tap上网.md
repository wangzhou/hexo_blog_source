---
title: qemu虚拟机通过tun/tap上网
tags:
  - 虚拟化
  - 网络
  - QEMU
description: 在学习perf的时候，需要使的qemu虚拟机可以上网，下面是上网的设置过程
abbrlink: a95aba1
date: 2021-07-17 11:17:00
categories:
---

参考: http://www.360doc.com/content/12/0611/14/7982302_217438857.shtml

考虑用tun/tap的方式，那么需要宿主机(本人的是ubuntu12.04)的内核支持tun/tap的功能，
宿主机内核是支持这样的功能的，如果您用的操作系统内核不支持tun/tap，需要下载源码
然后编译对应的模块，然后插入对应的模块。

1) 主机已经有了/dev/net/tun, 需要修改可执行权限:
   sudo chmod o+x /dev/net/tun
   可以通过主机/boot/config-XXX文件查看是否配置了CONFIG_TUN, 2.6.X以后的内核
   已默认将TUN直接编译进内核，所以这里是CONFIG_TUN=y

2) 下载tunctl工具的源码：
   http://sourceforge.net/prjects/tunctl/
   解压之后直接make就好了。过程中有可能会出现make: docbook2man: Command not 
   found, 这个是在生成对应的帮助文档的时候缺少了docbook2man的工具，现在ls一
   下发现tunctl已经编译好了，这个错误可以不用去管。同样的道理make install也
   会出错，不用make install了，直接用tunctl就可以了

3) 参考[1]进行设置，要注意确定主机内核对TUN/TAP设备和虚拟网桥的支持，这里采用
   TUN/TAP的方式搭建网络。第一步已经确认了TUN/TAP，现在只需要确认网桥：
   思路是要先找到网桥的内核配置项, 然后去/boot/config-XXX下查看：
   随便一个内核源码，查看linux-src/net/bridge/Kconfig，其中第一项就是参考[1]
   中的网桥的说明，可见配置项就是CONFIG_BRIDGE; 在/boot/config-XXX中查看，发
   现CONFIG_BRIDGE=m, 于是用sudo modprobe bridge将其插入内核中

4) 将参考[1]中的配置在这里重复一下：
   ifconfig eth0 down                # 关闭网口
   brctl addbr br0                   # 添加虚拟的网桥
   brctl addif br0 eth0              # 在网桥上添加网口
   brctl stp br0 off                 
   brctl setfd br0 1                 
   brctl sethello br0 1              
   ifconfig br0 0.0.0.0 promisc up       # 释放br0的ip地址
   ifconfig eth0 0.0.0.0 promisc up      # 释放eth0的ip地址
   dhclient br0# 自动获得br0的ip地址
   brctl show br0
   brctl showstp br0

   tunctl -t tap0 -u root        # 设定虚拟网卡上的端口tap0
   brctl addif br0 tap0# 在网桥上添加虚拟网卡的端口tap0
   ifconfig tap0 0.0.0.0 promisc up# 释放tap0的ip地址
   brctl showstp br0

5) 启动qemu虚拟机：
   sudo qemu-system-x86_64 -m 1024 -net nic -net tap,ifname=tap0,script=no,downscript=no ./ubuntu_qemu.img
   这里假设虚拟机已经可以运转起来，关于ubuntu_qemu.img的制作可以查看qemu-img的用法

6) 测试虚拟机网络：
   ping www.baidu.com
   可以了

附录：
   虚拟机自己的ip是：192.168.201.108/25
   br0的ip是：       192.168.201.23/25
   tap0, eth0没有ip, 但是eth0, br0的MAC地址相同
   主从之间通过上面的ip可以相互通信，另一台独立的pc通过虚拟机ip可以和虚拟机通信
```
	+----------+     
	|bridge:   |---> br0 (ip:192.168.201.23/25 78:ac:c0:a8:fc:fe)
	|br0       |---> eth0(78:ac:c0:a8:fc:fe)
	|          |
	+----------+
	     |
	     |tap0(ca:fd:08:a3:65:9d)
	     |
	     |eth0(192.168.201.108/25 52:54:00:12:34:56)
	+-----------+
	| guest os  |
	|           |
	+-----------+
```
