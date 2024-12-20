---
title: riscv qemu virt平台外设分析
tags:
  - QEMU
  - riscv
description: 本文分析riscv架构qemu virt虚拟机上各种外设的逻辑，分析基于qemu v7.1.50(d29201ff3)， Linux内核基于v5.19。
abbrlink: 64871
date: 2022-09-25 21:16:28
categories:
---

分析线索
---------

 riscv qemu virt上的各个设备的device tree节点生成和设备初始化在hw/riscv/virt.c里。
 基本逻辑是：
```
 /*
  * virt machine对象初始化函数, 这个函数的逻辑大概分三个部分：1. 创建相关外设；
  * 2. 生成外设的device tree节点；3. 添加virt_machine_done函数，这个函数用来加载
  * bios, 内核以及文件系统，准备开始运行bios之前的一段小程序。
  */
 virt_machine_init
       /* 创建核内中断控制器 */
   +-> riscv_aclint_swi_create/riscv_aclint_mtimer_create
       /* 创建外设中断控制器 */
   +-> virt_create_plic
       /* 创建内存和启动会用到的rom */
   +-> memory_region_add_subregion/memory_region_init_rom
       /*
        * 创建reboot和poweroff用的qemu设备，这个设备的逻辑比较简单，对外只呈现一个
        * 写接口，poweroff请求就是exit qemu进程，reboot就是调用qemu reset相关函数
        */
   +-> sifive_test_create
       /* 创建virtio-mmio设备 */
   +-> sysbus_create_simple("virtio-mmio", ...)
       /* 创建串口 */
   +-> serial_mm_init
       /* 创建RTC设备 */
   +-> sysbus_create_simple("goldfish_rtc", ...)
       /* 创建flash */
   +-> virt_flash_create/pflash_cfi01_legacy_drive/virt_flash_map
       /* 创建外设的device tree节点 */
   +-> create_fdt
       /* 这里只是注册一个回调函数 */
   +-> virt_machine_done
```
 我们看看怎么快速找见上面外设的qemu实现和Linux内核驱动。由于device tree描述的平台
 设备驱动和设备绑定是通过compatible字段的，只要在内核里搜compatible字段的名字就好。
 compatible的名字可以在create_fdt代码里得到，qemu支持只dump device tree，具体的
 命令是：
```
 qemu-system-riscv64 -machine virt -machine dumpdtb=rv.dtb
 dtc rv.dtb -I dtb -o rv.dts
```
 如果没有dtc这个工具，ubuntu上可以这样安装下：apt-get install device-tree-comiler

 在qemu monitor里使用info qtree可以列出虚拟机上有所得设备，比如：
```
  dev: fw_cfg_mem, id ""
    data_width = 8 (0x8)
    dma_enabled = true
    x-file-slots = 32 (0x20)
    acpi-mr-restore = true
    mmio 0000000010100008/0000000000000002
    mmio 0000000010100000/0000000000000008
    mmio 0000000010100010/0000000000000008
  [...]
```
 其中dev这行的第一个域段是这个设备的名字，比如，如上fw_cfg_mem，我们就可以发现，
 它的qemu驱动在hw/nvram/fw_cfg.c, 这个设备的名字定义在：
```
 /* hw/nvram/fw_cfg.c */
 static const TypeInfo fw_cfg_mem_info = {                                       
     .name          = TYPE_FW_CFG_MEM,    <--- fw_cfg_mem的宏定义
     .parent        = TYPE_FW_CFG,                                               
     .instance_size = sizeof(FWCfgMemState),                                     
     .class_init    = fw_cfg_mem_class_init,                                     
 };                                                                              
```

逐个设备分析
-------------

 qemu v7.1.50中，riscv virt平台上的设备有：fw_cfg_mem, cfi.pflash01, goldfish_rtc,
 serial-mm, pci相关, virtio-mmio, riscv.sifive.test, riscv.sifive.plic, riscv.aclint.mtimer,
 riscv.aclint.swi, riscv.hart_array。

 除了这些设备，还有PMU，因为是在核上，qemu的外设没有列出来。下面我们只是梳理下外设
 的大概逻辑，不过详细的分析。

 - fw_cfg_mem

   这个是qemu里定义的一个虚拟设备, 在qemu代码docs/specs/fw_cfg.rst有它的接口描述。
   通过qemu启动命令可以把信息写入这个设备，系统上，通过内核驱动可以对这个设备做
   读写以及配置操作。

 - cfi.pflash01

   系统里有两个该类型的flash。

 - goldfish_rtc

   一个提供墙上时钟的RTC设备。

 - serial-mm

   经典的8250串口。

 - pci相关

   提供PCI root port，其他的PCI/PCIe设备可以接入到这个root port下。

 - virtio-mmio

   提供多个virtio-mmio设备，但是似乎这些设备没有具体的业务功能。

 - riscv.sifive.test

   一个qemu中定义的设备，代码在hw/misc/sifive_test.c，这个设备很简单，就是提供了
   一个MMIO写接口，软件通过这个写接口可以触发qemu虚拟机的poweroff和reset, 其中
   poweroff就是exit掉qemu进程，reset是直接调用qemu_system_reset_request。

   这个设备和syscon-reboot/syscon-poweroff共同支持虚拟机里linux系统的reset和poweroff。

 - riscv.sifive.plic

   riscv系统上常用的核外外设中断控制器。 

 - riscv.aclint.mtimer

   核内M mode timer，就是产生时钟中断的设备，一个这样的设备会有一组mtime, mtimcmp
   寄存器，这两个寄存器是riscv特权级构架定义中的。

 - riscv.aclint.swi

   核内software interrupt的设备，是一组核上的寄存器抽象出的设备，每个核都有一组
   对应的寄存器，向相关寄存器写入数据触发其他核上的software interrupt，所以，Linux
   内核里用这个接口触发IPI。

   qemu 7.1.50上有两个这样的设备, 名字都叫riscv.aclint.swi，其中一个是sswi，另一个
   是mswi。

 - riscv.hart_array 
   
   相关分析细节可以看[这里](https://wangzhou.github.io/riscv-qemu-virt平台CPU拓扑分析/)

 - riscv pmu

   PMU设备，riscv核上用于做性能统计的寄存器组合起来抽象成这个设备。
