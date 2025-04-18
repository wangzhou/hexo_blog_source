---
title: 如何在qemu里增加一个虚拟设备
tags:
  - QEMU
  - 虚拟化
description: >-
  本文介绍在qemu里增加一个虚拟设备的步骤。本文会以一个PCIe DMA engine设备为
  例来介绍，我们定义这个设备的软硬件接口，并且按照这样的定义在qemu里实现这个
  设备，最后我们实现这个设备的Linux内核驱动。使用qemu可以方便的调试Linux内核，
  有了自定义的qemu设备，使用qemu调试与设备有关的问题也会变得比较方便。
abbrlink: 6555a03f
date: 2021-09-17 22:56:05
categories:
---

一个简易PCIe DMA engine设备的定义
---------------------------------

 内存拷贝比较消耗CPU资源，定义一个专用的DMA设备帮助CPU做内存拷贝，CPU把数据的地址
 和需要拷贝到的目的地址，配置到这个设备的相关寄存器里，然后启动数据拷贝，数据拷贝
 完成后，设备的相关寄存器的值改变，从而向软件报告任务完成。也可以通过设备中断的
 方式向软件报告任务完成。

 本文选DMA engine设备，完全和具体业务没有关系，只是因为这个业务模型比较简单，容易
 说明问题。

 具体数据搬移的实现也很简单：先把数据搬移到这个设备内部的buffer里，然后再把buffer
 里的数据搬移到目的地址。

 这个搬移的过程可以过IOMMU设备，也可以不过IOMMU，我们可以控制qemu系统，使得被模拟
 的平台有IOMMU设备或者没有IOMMU设备。

 如下是这个设备MMIO寄存器的具体定义:

 这个设备有一个32bit non-prefetch BAR, BAR base + 0x1000的位置是我们定义的寄存器。

| offset    |   定义                                                 |
|:--------- |:------------------------------------------------------ |
|0x0(只写)  |   原数据的地址。                                       |
|0x8(只写)  |   搬移的目的地址。                                     |
|0x10(只写) |   0-31 bit保留，32-63 bit表示搬移数据的长度。          |
|0x18(读写) |   bit0 置1表示开始拷贝数据。bit32 置1表示数据拷贝完成。|


qemu设备的实现
--------------

- 加入qemu编译系统

  本文用来arm64平台的qemu虚拟机测试。首先我们要保证整个平台的编译运行。测试使用
  的qemu的版本是6.1.0

  我们用如下的命令编译和启动基础的qemu虚拟机：
```
configure --target-list=aarch64-softmmu --enable-kvm
make -j64

qemu-system-aarch64 -cpu cortex-a57 \
-smp 1 -m 1024M \
-nographic \
-machine virt,iommu=smmuv3 \
-kernel ~/Image \
-append "console=ttyAMA0 earlycon root=/dev/ram rdinit=/init" \
-initrd ~/rootfs.cpio.gz \
```
  其中，如果想去掉smmuv3这个设备可以把machine那一行改成-machine virt \

  qemu使用meson来编译，我们只要在对应的meson.build文件中加入我们要编译的文件就好。
  把这个DMA engine设备的代码放到qemu/hw/misc/dma_engine.c里，所以在qemu/hw/misc/meson.build
  里加入如下的代码:
```
softmmu_ss.add(when: 'CONFIG_DMA_ENGINE', if_true: files('dma_engine.c'))
```
  表示如果配置CONFIG_DMA_ENGINE打开，就把dma_engine.c编译进来。

  在qemu/hw/misc/Kconfig里加入CONFIG_DMA_ENGINE，并配置把他直接编译到qemu里：
```
config DMA_ENGINE
    bool
    default y
```
  完成如上的配置后，我们在hw/misc目录加的dma_engine.c就可以参加到qemu编译里来。
  
- 定义一个PCIe设备
  
  如下[1]中是DMA engine PCIe设备的代码，我们可以套用这个模板创建其他的PCIe设备。
  改变PCIDeviceClass里的vendor_id/device_id/revision/class_id的数值，虚拟设备
  PCIe配置空间中的对应域段的值可以被改变。

  class中的realize函数和instance_init函数是主要要实现函数。可以使用pci_register_bar
  来给这个PCIe设备增加BAR，通过pcie_xxxxx_cap_init来给这个设备增加PCIe的各种capability。

  qemu里对PCI和PCIe设备是分开模拟的，如果你要加PCIe设备相关的capability，需要
  创建一个PCIe设备，这个需要interfaces定义成 INTERFACE_PCIE_DEVICE，以及为这个
  设备加上PCIe extend capability，使用pcie_endpoint_cap_init就可以了。我们这里的
  DMA engine就是一个PCIe设备。在qemu里需要通过一个PCIe RP把一个PCIe设备接入到系统
  里。下面的章节会提到具体的qemu命令。

  模拟设备的代码里，用DmaEngState表示被模拟的设备，因为他是一个PCIe设备，所以在
  这个结构体一开始的位置放一个PCIDevice的结构，后面的DmaRawState的结构用来放和具体
  业务相关的东西。

  使用lspci可以看到我们模拟出的是这样一个PCIe设备：
```
# ./lspci -s 00:04.0 -vvv
00:04.0 Class 00ff: Device 1234:3456 (rev 10)
        Subsystem: Device 1af4:1100
        Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR- FastB2B- DisINTx+
        Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
        Latency: 0
        Interrupt: pin A routed to IRQ 55
        IOMMU group: 0
        Region 0: Memory at 10260000 (32-bit, non-prefetchable) [size=16K]
        Capabilities: [e0] Express (v2) Root Complex Integrated Endpoint, MSI 00
                DevCap: MaxPayload 128 bytes, PhantFunc 0
                        ExtTag- RBE+ FLReset-
                DevCtl: CorrErr- NonFatalErr- FatalErr- UnsupReq-
                        RlxdOrd- ExtTag- PhantFunc- AuxPwr- NoSnoop-
                        MaxPayload 128 bytes, MaxReadReq 128 bytes
                DevSta: CorrErr- NonFatalErr- FatalErr- UnsupReq- AuxPwr- TransPend-
                DevCap2: Completion Timeout: Not Supported, TimeoutDis- NROPrPrP- LTR-
                         10BitTagComp- 10BitTagReq- OBFF Not Supported, ExtFmt+ EETLPPrefix+, MaxEETLPPrefixes 4
                         EmergencyPowerReduction Not Supported, EmergencyPowerReductionInit-
                         FRS-
                         AtomicOpsCap: 32bit- 64bit- 128bitCAS-
                DevCtl2: Completion Timeout: 50us to 50ms, TimeoutDis- LTR- 10BitTagReq- OBFF Disabled,
                         AtomicOpsCtl: ReqEn-
        Capabilities: [40] MSI: Enable+ Count=1/1 Maskable- 64bit+
                Address: 00000000fffff040  Data: 0050
        Kernel driver in use: dma_engine
```

- MMIO

  DmaRawState里的MemoryRegion用来表示这个模拟设备上BAR对应的MMIO。在realize函数里
  把这个地址空间和BAR0绑定，在dma_engine_state_init函数里为这段MMIO挂上read/write
  处理函数。read/write的回调函数定义在dma_engine_io_ops, 软件读写相关的寄存器最终
  都会在这些回调函数中处理。可以看到我们BAR size配置成了16KB。

- DMA

  软件写0x1018 bit0为1触发一个DMA数据拷贝，相关实现代码在dma_engine_do_dma_copy里。
  这个函数直接调用pci_dma_read/pci_dma_write做设备buffer和内存之间的数据搬移，这里
  在使能smmuv3的时候，以上两个函数的内部实现会先调用smmuv3的translation函数做地址
  翻译，然后再做数据搬移，在不使能smmuv3的时候，直接使用内存地址做数据搬移。

  如果只是模拟一个PCIe EP设备，可以不用管和iommu相关的内部实现。如果要做iommu的模拟，
  可以参考[这里](https://wangzhou.github.io/qemu-iommu模拟思路分析/)

- 中断

  realize函数中使用pci_config_set_interrupt_pin给设备加一个INTx中断。使用msi_init
  给设备加MSI中断。可以使用pci_irq_assert触发一个电平中断，通过msi_notify触发一个
  MSI中断，通过qemu_irq_pulse触发一个边沿中断。

Linux内核驱动的实现
-------------------

 DMA engine设备的内核驱动比较简单，源码在[2]。这个驱动同时也通过sysfs接口向用户态
 暴露了一个叫copy_size的文件，向这个文件写入一个数值将触发一次该数值大小的DMA数据
 拷贝，为了方便起见，我们在这个驱动的内部生成需要拷贝的数据。

编译运行
--------

 qemu的编译运行在上面有提及。使用的Linux内核的基础版本是5.15-rc1，编译的时候打开
 ARM_SMMU_V3和DMA_ENGINE_DEMO的配置即可。

 我们可以在启动qemu的时候带上--trace "smmuv3_*"，这样可以打开qemu smmuv3的trace，
 观察到smmuv3这个模拟设备内部的详细运行情况。我们也可以给DMA engine这个设备用类似
 的方法加上trace。
```
# echo 128 > copy_size 
[  161.037614] dma_engine 0000:00:04.0: input size: 80
smmuv3_config_cache_miss Config cache MISS for sid=0x20 (hits=0, misses=1, hit rate=0)
smmuv3_find_ste sid=0x20 features:0x1, sid_split:0x8
smmuv3_find_ste_2lvl strtab_base:0x4000000042c79000 l1ptr:0x42c79000 l1_off:0x0, l2ptr:0x7cc60000 l2_off:0x20 max_l2_ste:511
smmuv3_get_ste STE addr: 0x7cc60800
smmuv3_get_cd CD addr: 0x43bb4000
smmuv3_decode_cd oas=44
smmuv3_decode_cd_tt TT[0]:tsz:16 ttb:0x43dd5000 granule_sz:12 had:0
smmuv3_translate_success smmuv3-iommu-memory-region-32-4 sid=0x20 iova=0xffffe900 translated=0x427ff900 perm=0x1
smmuv3_config_cache_hit Config cache HIT for sid=0x20 (hits=1, misses=1, hit rate=50)
smmuv3_translate_success smmuv3-iommu-memory-region-32-4 sid=0x20 iova=0xffffe980 translated=0x427ff980 perm=0x2
smmuv3_cmdq_consume prod=42 cons=40 prod.wrap=0 cons.wrap=0
smmuv3_cmdq_opcode <--- SMMU_CMD_TLBI_NH_VA
smmuv3_s1_range_inval vmid=0 asid=1 addr=0xffffe000 tg=1 num_pages=0x1 ttl=3 leaf=1
smmuv3_cmdq_consume prod=42 cons=41 prod.wrap=0 cons.wrap=0
smmuv3_cmdq_opcode <--- SMMU_CMD_SYNC
smmuv3_cmdq_consume_out prod:42, cons:42, prod_wrap:0, cons_wrap:0 
smmuv3_write_mmio addr: 0x98 val:0x2a size: 0x4(0)
smmuv3_read_mmio addr: 0x9c val:0x2a size: 0x4(0)
[  161.043200] dma_engine 0000:00:04.0: dma engine test sucessed!
```
 如上的日志，smmuv3前缀的是qemu的trace，时间戳开头的是内核驱动的打印。

[1]https://github.com/wangzhou/qemu/blob/4612113da02716e8c56930e88ca8a142e180f175/hw/misc/dma_engine.c
[2]https://github.com/wangzhou/linux/blob/87695695e4d3ea72e60d9c5da5fc5804ae71fb48/drivers/misc/dma_engine/dma_engine.c
