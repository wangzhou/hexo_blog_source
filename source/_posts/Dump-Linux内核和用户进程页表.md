---
title: Dump Linux内核和用户进程页表
tags:
  - Linux内核
  - 内存管理
description: 本文介绍dump Linux内核和用户进程页表的办法
abbrlink: 177ea8b1
date: 2021-06-19 10:15:42
---

1. dump 内核页表

   1. 打开内核编译选项：CONFIG_PTDUMP_CORE, CONFIG_PTDUMP_DEBUGFS, 编译内核。

   2. 在/sys/kernel/debug/kernel_page_tables下可以dump kernel page table。dump
      出的数据大概如下:
```
0x0000000000000000-0xffff000000000000    16776960T PGD
0xffff000000000000-0xffff0000002dd000        2932K PTE       RW NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff0000002dd000-0xffff0000002de000           4K PTE       ro NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff0000002de000-0xffff00002d800000      742536K PTE       RW NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff00002d800000-0xffff00002f200000          26M PMD       ro NX SHD AF        BLK UXN    MEM/NORMAL
0xffff00002f200000-0xffff00002f3a0000        1664K PTE       ro NX SHD AF            UXN    MEM/NORMAL
0xffff00002f3a0000-0xffff0000385ec000      149808K PTE       RW NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff0000385ec000-0xffff0000386c0000         848K PTE
0xffff0000386c0000-0xffff0000386e0000         128K PTE       RW NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff0000386e0000-0xffff000038750000         448K PTE
0xffff000038750000-0xffff00003bc20000       54080K PTE       RW NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff00003bc20000-0xffff00003be00000        1920K PTE
0xffff00003be00000-0xffff00003c000000           2M PMD
0xffff00003c000000-0xffff000040000000          64M PTE       RW NX SHD AF            UXN    MEM/NORMAL-TAGGED
0xffff000040000000-0xffff008000000000         511G PUD
0xffff008000000000-0xffff800000000000      130560G PGD
---[ Linear Mapping end ]---
---[ BPF start ]---
0xffff800000000000-0xffff800000001000           4K PTE       ro x  SHD AF            UXN    MEM/NORMAL
0xffff800000001000-0xffff800000200000        2044K PTE
0xffff800000200000-0xffff800008000000         126M PMD
---[ BPF end ]---
---[ Modules start ]---
0xffff800008000000-0xffff800010000000         128M PMD
---[ Modules end ]---
---[ vmalloc() area ]---
0xffff800010000000-0xffff800011200000          18M PMD       ro x  SHD AF        BLK UXN    MEM/NORMAL
0xffff800011200000-0xffff800011240000         256K PTE       ro x  SHD AF    CON     UXN    MEM/NORMAL
0xffff800011240000-0xffff800011400000        1792K PTE       ro NX SHD AF            UXN    MEM/NORMAL
0xffff800011400000-0xffff800011a00000           6M PMD       ro NX SHD AF        BLK UXN    MEM/NORMAL
0xffff800011a00000-0xffff800011ba0000        1664K PTE       ro NX SHD AF            UXN    MEM/NORMAL
0xffff800011ba0000-0xffff800011e00000        2432K PTE
0xffff800011e00000-0xffff800012400000           6M PMD
0xffff800012400000-0xffff8000125d0000        1856K PTE
0xffff8000125d0000-0xffff800012600000         192K PTE       RW NX SHD AF    CON     UXN    MEM/NORMAL
0xffff800012600000-0xffff800012a00000           4M PMD       RW NX SHD AF        BLK UXN    MEM/NORMAL
0xffff800012a00000-0xffff800012b90000        1600K PTE       RW NX SHD AF    CON     UXN    MEM/NORMAL
0xffff800012b90000-0xffff800012b91000           4K PTE
0xffff800012b91000-0xffff800012b92000           4K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012b92000-0xffff800012b93000           4K PTE
0xffff800012b93000-0xffff800012b94000           4K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012b94000-0xffff800012b95000           4K PTE
0xffff800012b95000-0xffff800012b96000           4K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012b96000-0xffff800012b98000           8K PTE
0xffff800012b98000-0xffff800012b9c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012b9c000-0xffff800012b9d000           4K PTE
0xffff800012b9d000-0xffff800012b9e000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800012b9e000-0xffff800012ba0000           8K PTE
0xffff800012ba0000-0xffff800012bb0000          64K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800012bb0000-0xffff800012bb1000           4K PTE
0xffff800012bb1000-0xffff800012bb2000           4K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bb2000-0xffff800012bb3000           4K PTE
0xffff800012bb3000-0xffff800012bb4000           4K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bb4000-0xffff800012bb8000          16K PTE
0xffff800012bb8000-0xffff800012bbc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bbc000-0xffff800012bc0000          16K PTE
0xffff800012bc0000-0xffff800012bd0000          64K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800012bd0000-0xffff800012bd8000          32K PTE
0xffff800012bd8000-0xffff800012bdc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bdc000-0xffff800012be0000          16K PTE
0xffff800012be0000-0xffff800012be4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012be4000-0xffff800012be8000          16K PTE
0xffff800012be8000-0xffff800012bec000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bec000-0xffff800012bf0000          16K PTE
0xffff800012bf0000-0xffff800012bf4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bf4000-0xffff800012bf8000          16K PTE
0xffff800012bf8000-0xffff800012bfc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012bfc000-0xffff800012c00000          16K PTE
0xffff800012c00000-0xffff800012c04000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c04000-0xffff800012c08000          16K PTE
0xffff800012c08000-0xffff800012c0c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c0c000-0xffff800012c10000          16K PTE
0xffff800012c10000-0xffff800012c14000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c14000-0xffff800012c18000          16K PTE
0xffff800012c18000-0xffff800012c1c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c1c000-0xffff800012c20000          16K PTE
0xffff800012c20000-0xffff800012c24000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c24000-0xffff800012c28000          16K PTE
0xffff800012c28000-0xffff800012c2c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c2c000-0xffff800012c30000          16K PTE
0xffff800012c30000-0xffff800012c34000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c34000-0xffff800012c38000          16K PTE
0xffff800012c38000-0xffff800012c3c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c3c000-0xffff800012c40000          16K PTE
0xffff800012c40000-0xffff800012c44000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c44000-0xffff800012c48000          16K PTE
0xffff800012c48000-0xffff800012c4c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c4c000-0xffff800012c50000          16K PTE
0xffff800012c50000-0xffff800012c54000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012c54000-0xffff800012c55000           4K PTE
0xffff800012c55000-0xffff800012c75000         128K PTE       RW NX SHD AF            UXN    MEM/NORMAL-NC
0xffff800012c75000-0xffff800012c76000           4K PTE
0xffff800012c76000-0xffff800012c96000         128K PTE       RW NX SHD AF            UXN    MEM/NORMAL-NC
0xffff800012c96000-0xffff800012c97000           4K PTE
0xffff800012c97000-0xffff800012cb7000         128K PTE       RW NX SHD AF            UXN    MEM/NORMAL-NC
0xffff800012cb7000-0xffff800012cb8000           4K PTE
0xffff800012cb8000-0xffff800012cbc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cbc000-0xffff800012cc0000          16K PTE
0xffff800012cc0000-0xffff800012cc4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cc4000-0xffff800012cc8000          16K PTE
0xffff800012cc8000-0xffff800012ccc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012ccc000-0xffff800012cd0000          16K PTE
0xffff800012cd0000-0xffff800012cd4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cd4000-0xffff800012cd8000          16K PTE
0xffff800012cd8000-0xffff800012cdc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cdc000-0xffff800012ce0000          16K PTE
0xffff800012ce0000-0xffff800012ce4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012ce4000-0xffff800012ce8000          16K PTE
0xffff800012ce8000-0xffff800012cec000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cec000-0xffff800012cf0000          16K PTE
0xffff800012cf0000-0xffff800012cf4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cf4000-0xffff800012cf8000          16K PTE
0xffff800012cf8000-0xffff800012cfc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800012cfc000-0xffff800013000000        3088K PTE
0xffff800013000000-0xffff800013f60000       15744K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013f60000-0xffff800013fb0000         320K PTE
0xffff800013fb0000-0xffff800013fb4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fb4000-0xffff800013fbd000          36K PTE
0xffff800013fbd000-0xffff800013fbe000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fbe000-0xffff800013fc0000           8K PTE
0xffff800013fc0000-0xffff800013fc4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fc4000-0xffff800013fc5000           4K PTE
0xffff800013fc5000-0xffff800013fc6000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fc6000-0xffff800013fc8000           8K PTE
0xffff800013fc8000-0xffff800013fcc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fcc000-0xffff800013fcd000           4K PTE
0xffff800013fcd000-0xffff800013fce000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fce000-0xffff800013fd0000           8K PTE
0xffff800013fd0000-0xffff800013fd4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fd4000-0xffff800013fd5000           4K PTE
0xffff800013fd5000-0xffff800013fd6000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fd6000-0xffff800013fd8000           8K PTE
0xffff800013fd8000-0xffff800013fdc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fdc000-0xffff800013fdd000           4K PTE
0xffff800013fdd000-0xffff800013fde000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fde000-0xffff800013fe0000           8K PTE
0xffff800013fe0000-0xffff800013fe4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fe4000-0xffff800013fe5000           4K PTE
0xffff800013fe5000-0xffff800013fe6000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fe6000-0xffff800013fe8000           8K PTE
0xffff800013fe8000-0xffff800013fec000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013fec000-0xffff800013fed000           4K PTE
0xffff800013fed000-0xffff800013fee000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013fee000-0xffff800013ff0000           8K PTE
0xffff800013ff0000-0xffff800013ff4000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013ff4000-0xffff800013ff5000           4K PTE
0xffff800013ff5000-0xffff800013ff6000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013ff6000-0xffff800013ff8000           8K PTE
0xffff800013ff8000-0xffff800013ffc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800013ffc000-0xffff800013ffd000           4K PTE
0xffff800013ffd000-0xffff800013ffe000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800013ffe000-0xffff800014000000           8K PTE
0xffff800014000000-0xffff800014004000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800014004000-0xffff800014005000           4K PTE
0xffff800014005000-0xffff800014006000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800014006000-0xffff80001400d000          28K PTE
0xffff80001400d000-0xffff80001400e000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff80001400e000-0xffff800014015000          28K PTE
0xffff800014015000-0xffff800014016000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xffff800014016000-0xffff800014038000         136K PTE
0xffff800014038000-0xffff80001403c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff80001403c000-0xffff800014040000          16K PTE
0xffff800014040000-0xffff800014044000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800014044000-0xffff800014048000          16K PTE
0xffff800014048000-0xffff80001404c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff80001404c000-0xffff800014050000          16K PTE
0xffff800014050000-0xffff800014054000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800014054000-0xffff800014058000          16K PTE
0xffff800014058000-0xffff80001405c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff80001405c000-0xffff800014060000          16K PTE
0xffff800014060000-0xffff800014064000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800014064000-0xffff800014065000           4K PTE
0xffff800014065000-0xffff8000140a7000         264K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff8000140a7000-0xffff8000140a8000           4K PTE
0xffff8000140a8000-0xffff8000140b3000          44K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff8000140b3000-0xffff8000140b8000          20K PTE
0xffff8000140b8000-0xffff8000140bc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff8000140bc000-0xffff8000140c8000          48K PTE
0xffff8000140c8000-0xffff8000140cc000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff8000140cc000-0xffff8000140e8000         112K PTE
0xffff8000140e8000-0xffff8000140ec000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff8000140ec000-0xffff800014128000         240K PTE
0xffff800014128000-0xffff80001412c000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff80001412c000-0xffff800014130000          16K PTE
0xffff800014130000-0xffff800014134000          16K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800014134000-0xffff800014135000           4K PTE
0xffff800014135000-0xffff800014138000          12K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xffff800014138000-0xffff800014200000         800K PTE
0xffff800014200000-0xffff800020000000         190M PMD
0xffff800020000000-0xffff800030000000         256M PTE       RW NX SHD AF            UXN    DEVICE/nGnRnE
0xffff800030000000-0xffff800040000000         256M PMD
0xffff800040000000-0xffff808000000000         511G PUD
0xffff808000000000-0xfffffd8000000000         125T PGD
0xfffffd8000000000-0xfffffdff80000000         510G PUD
0xfffffdff80000000-0xfffffdffbfe00000        1022M PMD
0xfffffdffbfe00000-0xfffffdffbffd1000        1860K PTE
0xfffffdffbffd1000-0xfffffdffbffd4000          12K PTE       RW NX SHD AF            UXN    MEM/NORMAL
0xfffffdffbffd4000-0xfffffdffbfff0000         112K PTE
---[ vmalloc() end ]---
0xfffffdffbfff0000-0xfffffdffc0000000          64K PTE
0xfffffdffc0000000-0xfffffdfffe400000         996M PMD
0xfffffdfffe400000-0xfffffdfffe5f9000        2020K PTE
---[ Fixmap start ]---
0xfffffdfffe5f9000-0xfffffdfffe5fa000           4K PTE
0xfffffdfffe5fa000-0xfffffdfffe5fb000           4K PTE       ro x  SHD AF            UXN    MEM/NORMAL
0xfffffdfffe5fb000-0xfffffdfffe5fc000           4K PTE       ro NX SHD AF            UXN    MEM/NORMAL
0xfffffdfffe5fc000-0xfffffdfffe5ff000          12K PTE
0xfffffdfffe5ff000-0xfffffdfffe600000           4K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xfffffdfffe600000-0xfffffdfffe800000           2M PMD       ro NX SHD AF        BLK UXN    MEM/NORMAL
0xfffffdfffe800000-0xfffffdfffea00000           2M PMD
---[ Fixmap end ]---
0xfffffdfffea00000-0xfffffdfffec00000           2M PMD
---[ PCI I/O start ]---
0xfffffdfffec00000-0xfffffdfffec10000          64K PTE       RW NX SHD AF            UXN    DEVICE/nGnRE
0xfffffdfffec10000-0xfffffdfffee00000        1984K PTE
0xfffffdfffee00000-0xfffffdffffc00000          14M PMD
---[ PCI I/O end ]---
0xfffffdffffc00000-0xfffffdffffe00000           2M PMD
---[ vmemmap start ]---
0xfffffdffffe00000-0xfffffe0000e00000          16M PMD       RW NX SHD AF        BLK UXN    MEM/NORMAL
0xfffffe0000e00000-0xfffffe0040000000        1010M PMD
0xfffffe0040000000-0xfffffe8000000000         511G PUD
0xfffffe8000000000-0x0000000000000000        1536G PGD
```
2. dump 进程页表
   
   linux系统上，通过对/proc/\<pid\>/pagemap的操作可以dump进程的页表信息。
   这个文件的介绍可以参考linux/Documentation/vm/pagemap.txt。直接cat
   这个文件的无法得到信息，我们需要按照上面文件中描述的格式写代码解析。
   内核代码在tools/vm/page-types.c提供了解析的工具。可以使用:
   make -C tools/vm 编译得到page-types这个工具。使用的效果类似:
```
sudo ./page-types --pid 91228
             flags	page-count       MB  symbolic-flags			long-symbolic-flags
0x0000000000000800	         1        0  ___________M________________________________	mmap
0x000000000000086c	       706        2  __RU_lA____M________________________________	referenced,uptodate,lru,active,mmap
0x0000000000005828	       520        2  ___U_l_____Ma_b_____________________________	uptodate,lru,mmap,anonymous,swapbacked
0x000000000000586c	         1        0  __RU_lA____Ma_b_____________________________	referenced,uptodate,lru,active,mmap,anonymous,swapbacked
             total	      1228        4
```
   使用 ./page-types --pid <pid> --list-each 可以看到具体va到pa的映射：
```
voffset	offset	flags
aaaab6b57	6e938	__RU_lA____M________________________________
aaaab6b58	aa3cb	__RU_lA____M________________________________
aaaab6b59	55172	__RU_lA____M________________________________
[...]
ffffc744c	a17e7	___U_lA____Ma_b_____________________________
ffffc744d	117a3a	___U_lA____Ma_b_____________________________
ffffc744e	109913	___U_lA____Ma_b_____________________________
ffffc744f	136574	__RU_lA____Ma_b_____________________________


             flags	page-count       MB  symbolic-flags			long-symbolic-flags
0x0000000000000800	         1        0  ___________M________________________________	mmap
0x0000000000000804	         1        0  __R________M________________________________	referenced,mmap
0x000000000004082c	       429        1  __RU_l_____M______u_________________________	referenced,uptodate,lru,mmap,unevictable
0x0000000000000868	         3        0  ___U_lA____M________________________________	uptodate,lru,active,mmap
0x000000000000086c	      2009        7  __RU_lA____M________________________________	referenced,uptodate,lru,active,mmap
0x0000000000005868	      1073        4  ___U_lA____Ma_b_____________________________	uptodate,lru,active,mmap,anonymous,swapbacked
0x000000000000586c	         1        0  __RU_lA____Ma_b_____________________________	referenced,uptodate,lru,active,mmap,anonymous,swapbacked
             total	      3517       13
```
  这里显示的是页之间的映射，所以还要乘上页大小，如果是4K页的话，上面的va到pa的
  映射就是：(左边是va, 右边是pa)
```
aaaab6b57000	6e938000
aaaab6b58000	aa3cb000
aaaab6b59000	55172000
[...]
ffffc744c000	a17e7000
ffffc744d000	117a3a000
ffffc744e000	109913000
ffffc744f000	136574000
```
