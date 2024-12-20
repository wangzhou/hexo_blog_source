---
title: RISCV DTS描述分析
tags:
  - riscv
  - dts
description: >-
  本文分析riscv的dts描述格式，分析用的是riscv qemu virt平台，我们将分析不同
  启动参数下生成的dts格式，重点分析CPU、中断控制器以及PCIe控制器的相关定义。
  依赖的qemu版本是v7.1.50，依赖的内核版本是v6.1。
abbrlink: 65257
date: 2023-02-06 19:16:02
categories:
---

背景介绍
---------

 我们使用如下的qemu命令可以直接dump virt平台的dts文件:
```
qemu-system-riscv64 -machine virt,aclint=on -machine dumpdtb=rv_dtb.dtb \
    -m 256M \
    -smp 8 \
    -numa node,nodeid=0,mem=128M,cpus=0-3 \
    -numa node,nodeid=1,mem=128M,cpus=4-7 \
    -kernel ~/repos/linux/arch/riscv/boot/Image \
    -initrd ~/repos/buildroot/output/images/rootfs.cpio.gz
dtc rv_dtb.dtb -I dtb -o rv.dts
```
 我们可以用如上的numa配置观察在numa系统下生成的dts。riscv virt qemu支持不同的中断
 控制器配置，所有我们也可以把上面默认的plic中断控制器改成只用aplic、aplic-imsic，
 它们的配置分别是-machine virt,aia=aplic和-machine virt,aia=aplic-imsic，我们也
 把上面的aclint去掉，那么我们将用回clint。aclint也可以和aia的配置组合，比如可以用
 -machine virt,aclint=on,aia=aplic-imsic表示同时使用aclint和aplic-imsic。

 我们生成一个最复杂的numa + aclint + aia的配置，相关的dts文件在[这里](https://github.com/wangzhou/tests/blob/1b9695163d95e74c244530fb4eca771fa25ddd08/rv_dts/rv.dts_with_2_numa_node)。

CPU相关dts节点
---------------
 
 可以看到每个CPU都有一个类似这样的描述节点:
```
 cpus {                                                                  
         #address-cells = <0x01>;                                        
         #size-cells = <0x00>;                                           
         timebase-frequency = <0x989680>;                                
                                                                         
         cpu@0 {                                                         
                 phandle = <0x0f>;                                       
                 numa-node-id = <0x00>;                                  
                 device_type = "cpu";                                    
                 reg = <0x00>;                                           
                 status = "okay";                                        
                 compatible = "riscv";                                   
                 riscv,isa = "rv64imafdch_zicsr_zifencei_zihintpause_zawrs_zba_zbb_zbc_zbs_smaia_ssaia_sstc";
                 mmu-type = "riscv,sv48";                                
                                                                         
                 interrupt-controller {                                  
                         #interrupt-cells = <0x01>;                      
                         interrupt-controller;                           
                         compatible = "riscv,cpu-intc";                  
                         phandle = <0x10>;                               
                 };                                                      
         };                                                              
```
 reg表示CPU相关寄存器的基地址? riscv,isa描述CPU支持的特性。mmu-type描述MMU支持的
 特性。CPU里的这个interrupt-controller描述CPU core的本地"中断控制器"，在CPU core
 看来，像software interrupt、timer interrupt以及external interrupt这样的中断输入
 都可以中断CPU core当前的执行，这里软件抽象一个本地"中断控制器"出来。

 CPU和numa相关的信息描述在cpu-map里，一个cluster表示一个numa节点，相关的CPU core
 写到cluster里。

 相关的内核代码分析:
```
 start_kernel
       /* arch/riscv/kernel/setup.c */
   +-> setup_arch
   | +-> parse_dtb
   | | +-> early_init_dt_scan
   | |       /*
   | |        * dts中memory节点的解析和添加memblock，这里会去掉0x80000000-0x80200000
   | |        * 这段BIOS占用的内存。
   | |        */
   | |   +-> early_init_dt_scan_memory
   | |     +-> early_init_dt_add_memory_arch
   | |       +-> memblock_add
   | |
   | |   /* 展开dtb的入口? */
   | +-> unflatten_device_tree
   | |   /* 解析riscv.isa字符串 */
   | +-> riscv_fill_hwcap
   |   /*
   |    * riscv上在arch/riscv/kernel/smpboot.c, 这里是CPU的numa拓扑的解析代码的
   |    * 入口。
   |    *
   |    * 如何和内核numa相关结构建立上联系？
   |    */
   +-> smp_prepare_boot_cpu
     +-> init_cpu_topology
       +-> parse_dt_topology
```
 
 plic内核代码分析可以参考这里[这里](https://wangzhou.github.io/riscv-plic基本逻辑分析/)。

中断相关dts节点
----------------
 
 除了上面的本地"中断控制器"，riscv的平台上还可能有PLIC/AIA等核外中断控制器。

 在只有plic时，plic的dts节点大概是这样的：
```
plic@c000000 {                                                  
        phandle = <0x12>;                                       
        numa-node-id = <0x00>;                                  
        riscv,ndev = <0x5f>;                                    
        reg = <0x00 0xc000000 0x00 0x600000>;                   
        interrupts-extended = <0x10 0x0b 0x10 0x09 0x0e 0x0b 0x0e 0x09 0x0c 0x0b 0x0c 0x09 0x0a 0x0b 0x0a 0x09>;
        interrupt-controller;                                   
        compatible = "sifive,plic-1.0.0\0riscv,plic0";          
        #address-cells = <0x00>;                                
        #interrupt-cells = <0x01>;                              
};                                                              
                                                                
plic@c600000 {                                                  
        phandle = <0x11>;                                       
        numa-node-id = <0x01>;                                  
        riscv,ndev = <0x5f>;                                    
        reg = <0x00 0xc600000 0x00 0x600000>;                   
        interrupts-extended = <0x08 0x0b 0x08 0x09 0x06 0x0b 0x06 0x09 0x04 0x0b 0x04 0x09 0x02 0x0b 0x02 0x09>;
        interrupt-controller;                                   
        compatible = "sifive,plic-1.0.0\0riscv,plic0";          
        #address-cells = <0x00>;                                
        #interrupt-cells = <0x01>;                              
};                                                              
```
 可见这样的情况下是一个numa节点配置一个plic，riscv.ndev是一个plic支持的输入中断
 个数，interrupts-extended描述plic和CPU的连接关系，具体可以这样看：
```
        interrupts-extended = <0x08 0x0b <--- 0x08是local irq controller的phandle，
                               0x08 0x09      后面是是接哪个中断的标号，0xb是M ext irq。
                               0x06 0x0b      0x9是S ext irq。
                               0x06 0x09
                               0x04 0x0b
                               0x04 0x09
                               0x02 0x0b
                               0x02 0x09>;
```
 可以看见interrupts-extended完整描述了plic的输出和CPU的关系，包括M mode和S mode
 的外部中断。这种情况的最大问题是, 在numa系统里会有多个plic，一个numa节点上的plic
 只能连接本numa节点上的CPU。plic和CPU的逻辑关系可以参考[这里](https://wangzhou.github.io/riscv-plic基本逻辑分析/)。

 如下是只有aplic的dts节点，这里只写了numa0上的节点：
```
aplic@d000000 {                                                 
        phandle = <0x14>;                                       
        numa-node-id = <0x00>;                                  
        riscv,num-sources = <0x60>;                             
        reg = <0x00 0xd000000 0x00 0x8000>;                     
        interrupts-extended = <0x10 0x09 0x0e 0x09 0x0c 0x09 0x0a 0x09>;
        interrupt-controller;                                   
        #interrupt-cells = <0x02>;                              
        compatible = "riscv,aplic";                             
};                                                              
                                                                
aplic@c000000 {                                                 
        phandle = <0x13>;                                       
        numa-node-id = <0x00>;                                  
        riscv,delegate = <0x14 0x01 0x60>;                      
        riscv,children = <0x14>;                                
        riscv,num-sources = <0x60>;                             
        reg = <0x00 0xc000000 0x00 0x8000>;                     
        interrupts-extended = <0x10 0x0b 0x0e 0x0b 0x0c 0x0b 0x0a 0x0b>;
        interrupt-controller;                                   
        #interrupt-cells = <0x02>;                              
        compatible = "riscv,aplic";                             
};                                                              
```
 可以见到如上的aplic把M mode和S mode描述到了两个独立的节点上，分析原因？

 如下是aplic-imsic的dts节点：
```
aplic@d000000 {                                                 
        phandle = <0x16>;                                       
        numa-node-id = <0x00>;                                  
        riscv,num-sources = <0x60>;                             
        reg = <0x00 0xd000000 0x00 0x8000>;                     
        msi-parent = <0x12>;                                    
        interrupt-controller;                                   
        #interrupt-cells = <0x02>;                              
        compatible = "riscv,aplic";                             
};                                                              
                                                                
aplic@c000000 {                                                 
        phandle = <0x15>;                                       
        numa-node-id = <0x00>;                                  
        riscv,delegate = <0x16 0x01 0x60>;                      
        riscv,children = <0x16>;                                
        riscv,num-sources = <0x60>;                             
        reg = <0x00 0xc000000 0x00 0x8000>;                     
        msi-parent = <0x11>;                                    
        interrupt-controller;                                   
        #interrupt-cells = <0x02>;                              
        compatible = "riscv,aplic";                             
};                                                              
                                                                
aplic@d008000 {                                                 
        phandle = <0x14>;                                       
        numa-node-id = <0x01>;                                  
        riscv,num-sources = <0x60>;                             
        reg = <0x00 0xd008000 0x00 0x8000>;                     
        msi-parent = <0x12>;                                    
        interrupt-controller;                                   
        #interrupt-cells = <0x02>;                              
        compatible = "riscv,aplic";                             
};                                                              
                                                                
aplic@c008000 {                                                 
        phandle = <0x13>;                                       
        numa-node-id = <0x01>;                                  
        riscv,delegate = <0x14 0x01 0x60>;                      
        riscv,children = <0x14>;                                
        riscv,num-sources = <0x60>;                             
        reg = <0x00 0xc008000 0x00 0x8000>;                     
        msi-parent = <0x11>;                                    
        interrupt-controller;                                   
        #interrupt-cells = <0x02>;                              
        compatible = "riscv,aplic";                             
};                                                              

imsics@28000000 {                                               
        phandle = <0x12>;                                       
        riscv,group-index-shift = <0x18>;                       
        riscv,group-index-bits = <0x01>;                        
        riscv,hart-index-bits = <0x02>;                         
        riscv,num-ids = <0xff>;                                 
        reg = <0x00 0x28000000 0x00 0x4000 0x00 0x29000000 0x00 0x4000>;
        interrupts-extended = <0x10 0x09 0x0e 0x09 0x0c 0x09 0x0a 0x09 0x08 0x09 0x06 0x09 0x04 0x09 0x02 0x09>;
        msi-controller;                                         
        interrupt-controller;                                   
        #interrupt-cells = <0x00>;                              
        compatible = "riscv,imsics";                            
};                                                              
                                                                
imsics@24000000 {                                               
        phandle = <0x11>;                                       
        riscv,group-index-shift = <0x18>;                       
        riscv,group-index-bits = <0x01>;                        
        riscv,hart-index-bits = <0x02>;                         
        riscv,num-ids = <0xff>;                                 
        reg = <0x00 0x24000000 0x00 0x4000 0x00 0x25000000 0x00 0x4000>;
        interrupts-extended = <0x10 0x0b 0x0e 0x0b 0x0c 0x0b 0x0a 0x0b 0x08 0x0b 0x06 0x0b 0x04 0x0b 0x02 0x0b>;
        msi-controller;                                         
        interrupt-controller;                                   
        #interrupt-cells = <0x00>;                              
        compatible = "riscv,imsics";                            
};                                                              
```
 上面列出了numa系统中完整的aplic/imsic的dts节点，有imsic的情况下，aplic接到了imsic
 上，除此之外，aplic的dts没有变化。系统中新加入两个imsics dts节点，注意这两个imsics
 节点并不是按照numa节点划分，而是按照CPU mode划分，S mode的aplic接入第一个imsics,
 M mode的plic接入第二个imsics，每个imsics分别接入每个CPU core的对应mode的外部中断。

 (todo: aplic的输出怎么转化成一个MSI给imsics的?)

timer相关dts节点
-----------------

 每个numa节点一个mtimer。stimer只是增加了相关的CSR寄存器，并没有抽象出一个timer
 设备，所以dts并没有stimer的节点。其实，从下面的内核代码可以看出，内核是不需要这
 里mtimer的节点信息的(todo: 原因分析)。
```
mtimer@2000000 {                                                
        numa-node-id = <0x00>;                                  
        interrupts-extended = <0x10 0x07 0x0e 0x07 0x0c 0x07 0x0a 0x07>;
        reg = <0x00 0x2007ff8 0x00 0x08 0x00 0x2000000 0x00 0x7ff8>;
        compatible = "riscv,aclint-mtimer";                     
};                                                              
                                                                
mtimer@2008000 {                                                
        numa-node-id = <0x01>;                                  
        interrupts-extended = <0x08 0x07 0x06 0x07 0x04 0x07 0x02 0x07>;
        reg = <0x00 0x200fff8 0x00 0x08 0x00 0x2008000 0x00 0x7ff8>;
        compatible = "riscv,aclint-mtimer";                     
};                                                              
```
 timer的内核代码分析可以参考[这里](https://wangzhou.github.io/riscv-timer的基本逻辑/)。

PCIe相关dts节点
----------------

 如下是PCIe控制器的dts节点，我们整理了下interrupt-map和ranges的输出格式，这样看起
 来更清楚一点：
```
pci@30000000 {                                                  
        interrupt-map-mask = <0x1800 0x00 0x00 0x07>;           
        interrupt-map = <0x00 0x00 0x00 0x01 0x14 0x20 0x04     
                         0x00 0x00 0x00 0x02 0x14 0x21 0x04     
                         0x00 0x00 0x00 0x03 0x14 0x22 0x04     
                         0x00 0x00 0x00 0x04 0x14 0x23 0x04     
                         0x800 0x00 0x00 0x01 0x14 0x21 0x04    
                         0x800 0x00 0x00 0x02 0x14 0x22 0x04    
                         0x800 0x00 0x00 0x03 0x14 0x23 0x04    
                         0x800 0x00 0x00 0x04 0x14 0x20 0x04    
                         0x1000 0x00 0x00 0x01 0x14 0x22 0x04   
                         0x1000 0x00 0x00 0x02 0x14 0x23 0x04   
                         0x1000 0x00 0x00 0x03 0x14 0x20 0x04   
                         0x1000 0x00 0x00 0x04 0x14 0x21 0x04   
                         0x1800 0x00 0x00 0x01 0x14 0x23 0x04   
                         0x1800 0x00 0x00 0x02 0x14 0x20 0x04   
                         0x1800 0x00 0x00 0x03 0x14 0x21 0x04   
                         0x1800 0x00 0x00 0x04 0x14 0x22 0x04>; 
        ranges = <0x1000000 0x00 0x00
		  0x00 0x3000000 0x00 0x10000        <--- IO空间
		  0x2000000
		  0x00 0x40000000
		  0x00 0x40000000 0x00 0x40000000    <--- MEM空间
		  0x3000000
		  0x04 0x00
		  0x04 0x00 0x04 0x00>;
        reg = <0x00 0x30000000 0x00 0x10000000>;     <--- ECAM空间
        msi-parent = <0x12>;                                    
        dma-coherent;                                           
        bus-range = <0x00 0xff>;                                
        linux,pci-domain = <0x00>;                              
        device_type = "pci";                                    
        compatible = "pci-host-ecam-generic";                   
        #size-cells = <0x02>;                                   
        #interrupt-cells = <0x01>;                              
        #address-cells = <0x03>;                                
};                                                              
```
 interrupt-map/interrupt-map-map是PCIe控制器INTx中断的配置，ranges是MEM/IO窗口的
 配置，reg是ECAM空间的配置。
