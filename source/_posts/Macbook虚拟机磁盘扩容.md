---
title: Macbook虚拟机磁盘扩容
tags:
  - 虚拟化
  - LVM
description: 本文记录MacBook使用中给虚拟机磁盘扩容的过程。
abbrlink: 2c0af50
date: 2021-11-03 23:59:04
categories:
---

本人使用的是MacBook air，CPU使用M1芯片，在上面安装了parallels虚拟机，在虚拟机里
安装了Ubuntu20.04 arm64版本。在使用过程中发现虚拟机的磁盘空间不够。于是想到可以
给虚拟机磁盘扩容。

具体步骤如下：

1. 先关闭虚拟机里的ubuntu系统。

2. 在虚拟机配置里有磁盘选项，在这个选项里把磁盘容量配置成96GB。之前是64GB。

   配置完成打开Ubuntu，发现磁盘的容量是变大了，但是可以使用的容量还是没有变动，
   还是32GB。
```
sherlock@m1:~/notes$ lsblk
NAME                      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0  48.9M  1 loop /snap/core18/2068
loop1                       7:1    0  48.9M  1 loop /snap/core18/2127
loop2                       7:2    0  94.8M  1 loop /snap/docker/1124
loop3                       7:3    0  65.1M  1 loop /snap/gtk-common-themes/1515
loop4                       7:4    0  59.6M  1 loop /snap/lxd/20330
loop5                       7:5    0  28.1M  1 loop /snap/snapd/12707
loop6                       7:6    0  28.1M  1 loop /snap/snapd/12886
loop7                       7:7    0    62M  1 loop /snap/lxd/21032
loop8                       7:8    0   1.2M  1 loop /snap/stress-ng/5933
loop9                       7:9    0   1.2M  1 loop /snap/stress-ng/6068
loop10                      7:10   0 104.3M  1 loop /snap/docker/800
sda                         8:0    0    96G  0 disk 
├─sda1                      8:1    0   512M  0 part /boot/efi
├─sda2                      8:2    0     1G  0 part /boot
└─sda3                      8:3    0  94.5G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:0    0  36.3G  0 lvm  /
sr0                        11:0    1  1024M  0 rom  
```
   注意，如上是随后配置好时的情况，当时是ubuntu--vg-ubuntu--lv的总大小是32GB。

   可以看出ubuntu系统安装的时候使用lvm，安装比较时间长了，当时的估计是选了lvm。

3. 使用lvm的相关工具可以查看lvm的具体配置情况。
```
sherlock@m1:~/notes$ sudo pvdisplay
[sudo] password for sherlock: 
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               ubuntu-vg
  PV Size               <94.50 GiB / not usable 0   
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              24191
  Free PE               14911
  Allocated PE          9280
  PV UUID               Ul0ZEe-A6d4-H6Q7-zAso-Zmqa-oTxz-R0jS7b
   
sherlock@m1:~/notes$ sudo vgdisplay 
  --- Volume group ---
  VG Name               ubuntu-vg
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  4
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                1
  Open LV               1
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <94.50 GiB
  PE Size               4.00 MiB
  Total PE              24191
  Alloc PE / Size       9280 / 36.25 GiB
  Free  PE / Size       14911 / <58.25 GiB
  VG UUID               0bjAYY-GG24-Nkv3-3Xng-OIY8-PLiM-ALwMbu
   
sherlock@m1:~/notes$ sudo lvdisplay 
  --- Logical volume ---
  LV Path                /dev/ubuntu-vg/ubuntu-lv
  LV Name                ubuntu-lv
  VG Name                ubuntu-vg
  LV UUID                m0cTmJ-CrAW-WOjG-0eTR-L19N-dkT1-t7o78d
  LV Write Access        read/write
  LV Creation host, time ubuntu-server, 2021-04-30 21:54:25 +0800
  LV Status              available
  # open                 1
  LV Size                36.25 GiB
  Current LE             9280
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
```
   lvm里有物理卷，卷组和逻辑卷的概念。一般的使用逻辑是先用物理卷格式化磁盘分区，
   多个或者一个物理卷可以组成一个卷组，再在卷组上创建逻辑卷。使用pvdisplay, vgdisplay,
   lvdisplay可以查看物理卷、卷组和逻辑卷的相关信息。如上是扩容后的信息，可以看到，
   我们在一个物理磁盘容量为96GB的磁盘上创建了一个96GB的pv，在这个pv上创建了一个
   96GB的vg，但是在这个vg上创建的lv只有36.25GB。扩容之前是32GB。

4. 明白这个底层逻辑后，我们只要搜索下lv扩容的命令就好。可以使用如下命令扩展lv。

   lvextend -L +4GB /dev/ubuntu-vg/ubuntu-lv

5. 用df -h看到上面逻辑卷的容量还没有扩大，这是因为df看的是文件系统的大小，还要对
   逻辑卷上的文件系统扩容下，我这里的文件系统是ext4。可以使用如下命令扩容。

   resize2fs /dev/ubuntu-vg/ubuntu-lv

Note: ubuntu下有基于图形的分区工具GParted
