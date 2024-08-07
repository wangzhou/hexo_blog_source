---
title: 如何尝试使用Linux SVA
tags:
  - Linux内核
  - SMMU
description: 本文介绍使用Linux SVA技术的方法，基于KunPeng920上的压缩解压缩设备。
abbrlink: 5cae2bae
date: 2021-06-21 13:05:38
---

 硬件确认
-----------

 首先你要有一台KunPeng920服务器，而且这台服务器上的压缩解压缩设备是可见的。你可以
 lspci -s 75:00.0 -vv
```
root@ubuntu:~# lspci -s 75:00.0
75:00.0 Processing accelerators: Device 19e5:a250 (rev 21)
```
 如上，说明你的系统上有这个压缩解压缩的设备。

 系统的SMMU要在UEFI里打开。你可以看下系统启动日志，dmesg | grep iommu
```
root@ubuntu:~# dmesg | grep iommu
[...]
[   19.410490] hisi_zip 0000:75:00.0: Adding to iommu group 14
[...]
```
 如上，可以认为SMMU的配置没有问题，当然group的编号可以是不同的。

内核配置和编译
-----------------

 目前内核的相关补丁还没有完全上主线，我们在Linaro的github上维护了一个完整的可以
 跑的分支：https://github.com/Linaro/linux-kernel-warpdrive/tree/uacce-devel

 make defconfig

 make menuconfig

 这里defconfig的配置是不够的，你需要确保如下的内核配置是打开的:
    CONFIG_ARM_SMMU_V3=y                                                            
    CONFIG_PCI_PASID=y                                                              
    CONFIG_IOMMU_SVA=y                                                              
    CONFIG_CRYPTO_DEV_HISI_QM=y                                                     
    CONFIG_CRYPTO_DEV_HISI_ZIP=y                                                    
    CONFIG_UACCE=y                                                                  
 
 然后编译内核即可。

用户态代码配置和编译
-----------------------

 对应的用户态代码的仓库也在Linaro的github上：https://github.com/Linaro/warpdrive/tree/master

 ./autogen.sh
 ./conf.sh
 make

 在.lib目录下会生成编译出的用户态库：
```
Sherlock@EstBuildSvr1:~/repos/linaro_wd/warpdrive/.libs$ ls *.so
libhisi_qm.so  libwd_ciper.so  libwd_comp.so  libwd_digest.so  libwd.so
```
 在test目录下有编译好的测试app：
```
 test_sva_bind test_sva_perf
```
 如上的两个测试app基于压缩解压缩设备，所以依赖的库是：
 libhisi_qm.so libwd_comp.so libwd.so

运行测试用例
---------------

 使用如上编译好的内核Image启动系统, 把libhisi_qm.so libwd_comp.so libwd.so
 拷贝到系统上，然后尝试运行下 test_sva_perf。如果运行OK的话会有性能数据打印出来：
```
root@ubuntu:/home/sherlock/warpdrive/test# ./test_sva_perf 
Compress bz=512000 nb=1×10, speed=1433.5 MB/s (±0.0% N=1) overall=1334.3 MB/s (±0.0%)
```

 test_sva_bind test_sva_perf里各个命令参数的用法可以参考help说明。

