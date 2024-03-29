---
title: ARM SMMUv3 architecture
tags:
  - Linux内核
  - SMMU
  - 计算机体系结构
description: 本文梳理IOMMU相关的整体软硬件设计的全貌。具体的硬件以ARM SMMUv3作为例子。
abbrlink: e1dd65ca
date: 2021-06-21 13:06:42
---

IOMMU是外设的MMU。原来的外设主动发起的DMA的操作使用的都是系统的物理地址，直接
使用物理地址有很多不方便的地方，在外设和内存之间引入一个新的IOMMU硬件，完成一些
诸如地址翻译的功能，这样又有很多新的玩法可以加进来。

加入IOMMU上的这个地址翻译，就可以引入翻译时候的权限管理，这样保证了外设发出的
访问系统内存的地址是安全的。加上地址翻译，还可以把一片连续的虚拟地址空间映射到
诸多离散的物理地址上，这样满足一部分设备要访问连续大地址的需要。IOMMU把外设
变得更像CPU，CPU和外设在使用内存方面都只看到虚拟地址, 如果虚拟地址到物理地址的
映射在CPU MMU和IOMMU上是一样的，CPU和外设将可以看到相同的虚拟地址空间。CPU MMU
上的七七八八的功能上都可以在IOMMU上都加上。

CPU core和MMU是绑在一起的，而IOMMU和外设是独立的两个设备, IOMMU一般是不做在
外设里的，不然带IOMMU的外设有了地址管理的功能，对系统不安全。MMU上地址翻译的
页表，不同进程切换的时候都要换一套, 外设并不能被进程独占，不同的进程可以同时
给外设发请求, 广义上作为外设状态一部分的IOMMU自然也用有别MMU的方法描述不同外设
在IOMMU的地址翻译配置。

为了说明白整个IOMMU/SMMU的大体架构, 大概要说清楚一下几个方面：

1. IOMMU(以下都用SMMU)是一个什么设备，它在系统中的位置和作用是什么。

2. 固件(UEFI)里如何描述SMMU和系统其他部件的关系(PCI, GIC), 如何描述SMMU和它
   管理的外设的关系。系统软件(一下都用Linux内核)如何解析，进而构建这种关系。

3. SMMU硬件都提供怎么样的功能, SMMU使用怎么样的软硬件结构来支持这样的功能。
   Linux内核如何构建SMMU硬件需要的执行环境。SMMU驱动运行时如何工作。

4. SMMU驱动怎么对外提供功能，外界访问IOMMU/SMMU的接口有哪些。IOMMU这一层如何支持
   IOMMU的对外接口。

5. SMMU的虚拟化(S1 + S2)是怎么用起来的。


下面来一一介绍下上面的内容。

SMMU的固件描述和Linux解析
----------------------------
  
  参考[2][3], ACPI(先不关注DT里的描述方法)在IORT表格里描述SMMU和系统里其他部件
  的关系。

SMMU功能简介
---------------

  SMMU为外设提供和地址翻译相关的诸多功能。硬件上, SMMU用三个队列和两个表格支持
  其基本功能。
```
               +----------+
               | CPU core |
               | +-----+  |
               | | MMU |  |
               +-+--+--+--+       +------------------------------------------+
   system bus       v             |                                          |
          ----------------------->|  DDR     +-------+                       |
                   ^              |          |pa     |                       |
             pa    |              |          +-------+                       |
                   |              |                                          |
               +---+-----+        | +---+    +---+    +----------+           |
               |  SMMU   |------->| |STE|--->|CD |--->|Page table|           |
               |         |        | |   |    +---+    | va -> ipa| (s1)      |
               |         |        | |   |    |.. |    +----------+           |
               |         |        | |   |    +---+                           |
               |  TLB    |        | |   |    |CD |                           |
               |         |        | |   |    +---+                           |
               |STE cache|<-------| |   |    +-----------+                   |
               |         |        | |   |--->|Page table | (s2)              |
               |CD cache |        | |   |    | ipa -> pa |                   |
               |         |        | +---+    +-----------+                   |
               |         |        | |.. |                                    |
               |         |        | +---+                                    |
               |         |        | |STE|                                    |
               |         |        | +---+                                    |
               |         |        |                                          |
               |         |        | +------------------+                     |
               |         |<-------| |command queue     |                     |
               |         |        | +------------------+                     |
               |         |        | +------------------+                     |
               |         |------->| |event queue       |                     |
               |         |        | +------------------+                     |
               |         |        | +------------------+                     |
               |         |------->| |pri queue         |                     |
               +---------+        | +------------------+                     |
                   ^              +------------------------------------------+
trasaction with va |              
           +---+       +---+      
           |dev|  ...  |dev|      
           +---+       +---+      
```
  如上是一个SMMU硬件的简单示意图, 为了把SMMU相关的一些内存里的数据结构也画下，
  上图把SMMU画的很大。一般的，一个SMMU物理硬件会同时服务几个外设，而每个外设又
  有可能可以独立的发出多个内存访问，这些内存访问需要靠SMMU相互区分，又要靠SMMU
  做地址翻译。SMMU硬件靠STE(stream table entry)和CD(context descriptor), 去区分
  不同硬件以及相同硬件上的不用内存访问流。如上STE和CD内存里的表格，SMMU硬件可以
  认知这些表格，一个外设相关的内存访问信息放在一个STE里，一个外设上的一部分资源
  的内存访问信息放在一个CD里。外设和STE的对应关系需要SID(stream id)建立联系，
  对于PCI设备，他的SID一般就是BDF，外设硬件在发出的内存访问请求中带上BDF信息，
  内存访问请求被SMMU解析，SMMU通过其中的SID找见对应的STE。外设的可以独立发内存
  访问请求的单位(比如外设的一个队列)和CD的关系需要SSID(substream id)建立联系，
  在PCI设备上，SSID对应的就是PCI协议里说的PASID，这个PASID一般从系统软件中申请
  得到，然后分别配置到外设的内存访问单元，和STE一样，设备发出内存访问的时候会
  带上这个PASID，SMMU根据PASID找见对应的CD。可以SMMU驱动需要先为对应的设备或者
  设备的独立内存访问单元建立STE或者CD，以及填充STE和CD中的域段以支持随后设备的
  内存访问。

  STE和CD里包含页表，和MMU一样，为了加快翻译速度，SMMU也做了TLB。为了加快STE和
  CD查找的速度，SMMU里也可能放STE和CD的cache。

  SMMU的软硬件控制接口，包括SMMU的基本的MMIO寄存器，三个硬件队列。如上，这是三个
  硬件队列中command queue用于软件向SMMU发送命令，event queue用于SMMU向软件报异常
  事件(包括缺页)，pri queue是和PCI设备配合一起用的，用于硬件向软件上报外设的
  page request请求。软件可以通过command queue向硬件发命令，SMMU的命令基本上可以
  分为，配置无效化命令，比如无效掉SMMU cache的STE和CD；TLB无效化命令，缺页相关
  的命令，比如用于继续stall请求的RESUME命令；prefetch命令；SYNC命令。
  

SMMU驱动分析
---------------

  SMMUv3相关的驱动的文件包括arm-smmu-v3.c, io-pgtable-arm.c,
  drivers/perf/arm_smmuv3_pmu.c。第一个文件是smmu驱动的主体，第二个文件是和
  页表相关的操作，第三个文件是和SMMU PMU(PMCG)相关的东西。第一二个文件编译出来
  SMMU驱动，第三个文件编译出SMMU PMCG驱动。

  SMMU驱动是一个普通的平台设备驱动。这驱动的probe函数里初始化SMMU硬件，包括：
  ACPI/DTS解析，中断初始化，硬件特性解析，SMMU STE初始化，probe里还把SMMU向
  iommu子系统注册，以及把iommu_ops回调函数注册给SMMU结构和总线。

  这其中涉及的SMMU驱动相关的一些数据结构包括：
```
  struct arm_smmu_device 描述一个物理的SMMU设备。
  struct fwnode_handle 描述struct iommu_device的固件描述。
  struct arm_smmu_master 描述SMMU物理设备所管理的一个外设, 这个外设可以对应一组
                         stream id, 但是一般是一个外设一个stream id。
  struct device
    +-> struct dev_iommu 一个外设device里和iommu相关的东西
      +-> struct iommu_fwspec 一个外设device和iommu硬件相关的东西
        +-> struct fwnode_handle iommu_device的固件描述
```
  probe里的流程比较直白：
```
  arm_smmu_device_probe
    ...
    +-> arm_smmu_device_hw_probe 探测各种硬件配置信息
    +-> arm_smmu_init_structures 初始化cmd, event, pri队列的内存, 初始化STE表,
                                 如果是两级STE表，只初始化L1 STE
    +-> iommu_device_register 向iommu子系统注册SMMU
    +-> arm_smmu_set_bus_ops 向pci，amba或者platform总线注册iommu_ops
```

  struct iommu_ops和具体iommu设备相关的回调函数需要通过上层的接口使用。
    +-> arm_smmu_add_device:
        创建外设对应的arm_smmu_master结构, 创建外设对应的iommu_group。
	外设和SMMU的关系怎么传给这个函数的？很明显在这个函数调用之前外设device
	之中的iommu_fwspece已经被赋值。这个赋值的地方在[1]中已经提到，就是
	pci_device_add(这里只看PCI设备的情况)，基本逻辑是PCI设备在被加入系统中
	的时候调用acpi_dma_configure找见它自己的RC，从而找见对应的SMMU。

	但是，内核代码后来修改了这部分，现在的内核代码(v5.7-rc1)，把xxx_dma_configure
	移到了really_probe里。已PCI总线为例，这里调用的是pci_dma_configure:

	pci_dma_configure
	  +-> acpi_dma_configure
	    +-> iort_iommu_configure 这个函数里创建device的iommu_fwspec
	    +-> arch_setup_dma_ops 调到比如arm64的回调中
	      +-> iommu_setup_dma_ops 在有iommu的情况下给device->dma_ops挂上iommu_dma_ops
	      (诸如dma_alloc_coherent的dma接口在做dma相关的操作时，在有iommu的
	       情况下，调用的就是这里挂在device->dma_ops里的各种回调函数)

	上面说的是为啥arm_smmu_add_device这个函数调用到的时候，device入参里已经
	有device->iommu_fwspec的域段，下面说arm_smmu_add_device这个函数怎么调用到：

	arm_smmu_add_device这个函数在iommu层里被封装, 然后注册成总线的一个notifier:
	smmu probe -> arm_smmu_set_bus_ops -> bus_set_iommu
	    +-> iommu_bus_init
	      +-> iommu_bus_notifier  BUS_NOTIFY_ADD_DEVICE会触发
	        +-> iommu_probe_device
		  +-> ops->add_device

	如上pci_device_add的最后会调用devcie_add向bus添加外设device, 这个过程
	会触发以上notifier回调函数，最终调用到arm_smmu_add_device。注意，这里
	really_probe在后，触发notifier执行在前。触发notifier在设备加载的时候，
	而really_probe在驱动加载的时候。

	下面看add_device都做了什么:
	add_device
	  +-> blocking_notifier_call_chain
	  +-> bus_probe_device
            +-> device_initial_probe
	      +-> __device_attach
	        +-> __device_attch_driver 注意：如果驱动没有加载不会再继续下面的调用
		...
		  +-> really_probe

	也就是说，一般先枚举设备，再insmod驱动的场景，只有在驱动加载的时候才会
	生成device结构里的iommu_fwspec。

	但是，实际跟踪执行流程，你会发现对于一个外设的arm_smmu_add_device调用是
	发生在对应的设备驱动加载的时候。这是因为smmu驱动加载比pci驱动晚，add_device
	想要触发notifier时，因为smmu驱动没有加载，notifier都还没有注册，自然也
	不会在add_device这个流程中触发arm_smmu_add_device这个函数。那么really_probe
	为啥会最终调用到arm_smmu_add_device? 这是因为在iort_iommu_configure里
	调用了iort_add_device_replay->iommu_probe_device。综上struct device里的
	和iommu有关的struct dev_iommu，以及iommu_fwspec都是在iort_iommu_configure
	里创建的。

	如最开始所述，这个arm_smmu_add_device创建特定外设在smmu处的各种描述信息。
	其中包括，创建arm_smmu_master, 建立设备对应的STE entry，初始化设备的
	pasid，pri功能，建立这个设备相关的iommu_group和iommu_domain, 在这个过程
	中会调用smmu ops里的device_group, domain_alloc以及attach_dev。

        架构设计上，iommu_group描述共享一个地址空间的设备的集合, iommu_domain
	打包地址翻译的类型以及为具体的iomm_domain实现提供桥梁，比如arm_smmu_domain。
	iommu_ops里的有些操作，比如，map会最终调用到arm_smmu_domain->io_pagetable_ops->map

	arm_smmu_add_device
	  +-> iommu_group_get_for_dev
	    +-> iommu_group_add_device
	      +-> __iommu_attach_device
	        +-> arm_smmu_attach_dev
	里创建iommu_group, iommu_domain, 并最终调用到arm_smmu_attach_dev。可以看到
	先按全局变量iommu_def_domain_type创建domain, 再用iommu_domain_set_attr
	补充设置domain的attr。一般domain的type有IDENTITY, UNMANAGED, DMA。IDENTITY
        是IOMMU不做处理，DMA是在内核里使用IOMMU，UNMANAGED是把IOMMU的能力暴露出去
	给其他模块使用，现在一般是VFIO和SVA再用UNMANAGED domain。attr用来区分
	domain里更细一步的配置，比如，arm_smmu_domain_set_attr里，对于DMA domain,
	用attr区分non_strict mode; 对于UNMANAGED domain, 用attr区分Nested的使用
	方式。可以看到UNMANAGED domain时, 如果只用一个stage，用的是stage1; 对于
	内核的DMA流程，一般顺着这个调用关系下来，但是，对于UNMANAGED domain和
	Nested，iommu还有其他对外接口，提供配置的方法。

	下面看arm_smmu_attach_dev里具体干了什么。

    +-> arm_smmu_attach_dev:
        这个函数的入参是，struct iommu_domain, struct device。如上，这里的domain
	表示的是一种使用方式。也就是说这里是把一个device和一种使用方式建立起联系。
	注意，上面的add_device是把一个外设加入到smmu这个系统里来。

	如上，我们可以看下iommu_domain的各种不同类型，以及内核里的不同子系统是
	怎么调用这个函数的，以及这个函数到底做了什么。

	这个函数对于特定的设备，根据传进来的domain类型, 初始化具体domain的相关
	操作函数，然后配置STE生效:
	arm_smmu_attach_dev
	  +-> arm_smmu_domain_finalise
	    +-> arm_smmu_domain_finalise_xxx[cd, s1, s2]
	    +-> smmu_domain->pgtbl_ops = alloc_io_pgtable_ops
	  +-> arm_smmu_install_ste_for_dev
	可见arm_smmu_domain_finalise里首先根绝smmu_domain里stage的配置，去配置
	smmu的CD,或者STE和s2相关的内容。然后，把对应的页表操作函数赋值给smmu_domain
	里的回调函数: smmu_domain->pgtbl_ops。以后，iommu_ops里的mmap，unmap函数
	直接会调用到这里。

	iommu_attach_group, iommu_attach_device, iommu_group_add_device,
	iommu_request_dm_for_dev, iommu_request_dma_domain_for_dev都会最终调用到
	attach_dev这个回调。在vfio里会使用到iommu_attach_group, iommu_attach_device,
	vfio会用UNMANAGED domain。(具体逻辑： todo)

IOMMU接口分析
----------------

  DMA接口和IOMMU的关系在上面已给出。

  目前社区正在上传SVA的相关补丁，SVA的接口也是IOMMU子系统对外接口的一部分, 这
  一部分的分析可以参见[5]。目前的SVA接口使用的是DMA domain, 通过SVA的接口触发
  在STE表里建立SSID非零的CD表，从而可以实现一个外设，一部分在内核态使用(使用CD0),
  一部风在用户态使用(使用其他CD)。

  VFIO重度使用IOMMU接口, 从这个角度看IOMMU子系统的对外接口参见[6]

SMMU虚拟化分析
-----------------

  Nested SMMU的分析参见[4]。

Reference

[1] https://wangzhou.github.io/PCI-SMMU-parse-in-ACPI/
[2] https://wangzhou.github.io/PCI-ACPI笔记2/
[3] https://wangzhou.github.io/PCI-SMMU-parse-in-ACPI/
[4] https://wangzhou.github.io/vSVA逻辑分析/
[5] https://wangzhou.github.io/Linux-SVA特性分析/
[6] https://wangzhou.github.io/Linux-vfio-driver-arch-analysis/
