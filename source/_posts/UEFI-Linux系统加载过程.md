---
title: UEFI/Linux系统加载过程
tags:
  - UEFI
description: 本文梳理一个UEFI加载linux启动的流程
abbrlink: c186a0ca
date: 2021-07-11 23:16:59
categories:
---

一般一个从硬盘启动的linux系统的启动顺序是：UEFI->GRUB->Linux。其中，我们一般称
UEFI是固件，GRUB是bootloader. 

顾名思义，一般我们认为固件是固化在系统里的，启动会自动加载、执行的一段代码，这
里我们不关心固件的存储位置。一般，bootloader和linux kernel镜像都是放在磁盘上
(我们这里只看磁盘启动的情况，不关心网络启动，e.g. PXE)。

UEFI在加载bootloader(e.g. grub)的时候会从EFI system分区寻找grub程序(e.g. grub.efi).
这个程序是一个UEFI环境中的可执行程序。一般，UEFI里会在EFI system分区上的一组路径
上搜索grub.efi，这组路径是提前在UEFI静态配置好的。EFI system分区必须被格式化成
FAT文件系统，这样UEFI才可以读取其中的文件。grub.efi加载后，会找见它对应的配置
文件grub.cfg. 在grub.cfg中，我们可以配置grub到哪里去加载kernel镜像, 以及到哪里去
加载根文件系统. 一个grub.cfg的配置类似:
```
# Sample GRUB configuration file
#
# Boot automatically after 5 secs.
set timeout=5
# By default, boot the Estuary with Centos filesystem
set default=d05_centos_sata_console
# For booting GNU/Linux

menuentry "D05 Centos SATA(CONSOLE)" --id d05_centos_sata_console {
        search --no-floppy --fs-uuid --set=root <UUID>
                linux /Image pcie_aspm=off pci=pcie_bus_perf rootwait root=PARTUUID=<PARTUUID> rw
}
```
其中上面的UUID是存放Image的分区的UUID, PARTUUID是存放根文件系统的分区的PARTUUID.
UUID, PARTUUID可以事先用blkid得到，比如：
```
[root@localhost ~]# blkid
/dev/sdc4: UUID="87b76c0a-7c76-4d2a-9414-6b52e6a00b1c" TYPE="xfs" PARTUUID="511f3dc7-4b6b-4991-975e-1d60e5e3e616" 
/dev/sdc2: UUID="2ee48246-43a2-4014-a176-5d723e6be5b4" TYPE="xfs" PARTUUID="c724693a-751a-40c7-a8bd-d3bdd56311e1" 
/dev/sdc1: SEC_TYPE="msdos" UUID="80A6-A4F2" TYPE="vfat" PARTLABEL="EFI System Partition" PARTUUID="f768b945-a14d-47d0-a5c0-371d4a163316" 
/dev/sdc3: UUID="417d3354-a5f8-4a2c-81e4-ef03b9d51e94" TYPE="swap" PARTUUID="d9ae07de-fdad-4fcc-8621-deeadace67c5" 
/dev/sdc5: UUID="c03a5dee-f751-4ff5-bc7c-f6026388a676" TYPE="xfs" PARTUUID="29860a29-e084-4105-815b-1a94c296a473" 
```

按照上面的配置做的系统只是一个临时可用的系统，在系统启动之后，如果不加入其他的
操作，启动分区(上面存放Image, grub.efi, grub.cfg)是没有办法挂在到根文件系统上的。
这样，如果我们想把内核编译成一个RPM包，然后用rpm -i kernel-package安装到系统，
使得下次系统启动的时候可以用新安装的内核，这样是不可能的。

rpm -i kernel-package的时候会把相关的配置加载grub.cfg里。to do...

实际上一个标准的ISO安装是这样的, 安装过成应该会自动的分区和在启动分区里放置相应
的文件，并且更改/etc/fstab里的内容，实现开机自动挂在启动分区到/boot. to do...


一个例子:

- 磁盘分区: fdisk [4]
- 拷贝grub.efi, grub.cfg, kernel Image到对应的EFI system分区
- 更改grub.cfg里的参数, 主要是修改上面提到的UUID和PARTUUID
- 拷贝文件系统到PARTUUID对应的分区里

reference:

[1] http://www.rodsbooks.com/linux-uefi/
[2] https://en.wikipedia.org/wiki/EFI_system_partition (install bootloader)
[3] http://www.rodsbooks.com/efi-bootloaders/installation.html
[4] https://github.com/open-estuary/estuary/blob/master/doc/Deploy_Manual.4D05.md#3.3
[4] https://github.com/open-estuary/estuary/blob/master/doc/Deploy_Manual.4D05.md#3.3
