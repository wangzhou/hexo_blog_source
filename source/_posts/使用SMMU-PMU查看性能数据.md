---
title: 使用SMMU PMU查看性能数据
tags:
  - Linux内核
  - perf
  - SMMU
description: >-
  ARM的SMMU提供了性能相关的统计寄存器(Performance Monitor Counter Groups - PMCG)，
  目前相关驱动已经合入Linux内核主线。我们可以配合用户态的perf工具使用。本文介绍具 体的使用方法。
abbrlink: 9d7a5764
date: 2021-06-19 10:30:16
---

- 首先要确定使用的系统里有arm_smmuv3_pmu这个模块，或者它已经被编译进内核。
  这个模块的代码在内核目录kernel/drivers/perf/arm_smmuv3_pmu.c, 内核配置是:
  CONFIG_ARM_SMMU_V3_PMU

- 确定使用的单板上的UEFI里有你要测试的模块对应的SMMU PMCG节点，没有这个节点的
  的话即使加载上面的驱动也无法使用SMMU PMCG

- 正常使用的话，dmesg | grep pmcg可以看见类似信息:
```
Ubuntu:/ # dmesg | grep pmcg
[ 1232.379951] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.8.auto: option mask 0x1
[ 1232.380040] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.8.auto: Registered PMU @ 0x0000000148020000 using 8 counters with Individual filter settings
[ 1232.380094] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.9.auto: option mask 0x1
[ 1232.380142] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.9.auto: Registered PMU @ 0x0000000201020000 using 8 counters with Individual filter settings
[ 1232.380190] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.10.auto: option mask 0x1
[ 1232.380241] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.10.auto: Registered PMU @ 0x0000000100020000 using 8 counters with Individual filter settings
[ 1232.380286] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.11.auto: option mask 0x1
[ 1232.380337] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.11.auto: Registered PMU @ 0x0000000140020000 using 8 counters with Individual filter settings
[ 1232.380397] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.12.auto: option mask 0x1
[ 1232.380445] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.12.auto: Registered PMU @ 0x0000200148020000 using 8 counters with Individual filter settings
[ 1232.380491] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.13.auto: option mask 0x1
[ 1232.380542] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.13.auto: Registered PMU @ 0x0000200201020000 using 8 counters with Individual filter settings
[ 1232.380601] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.14.auto: option mask 0x1
[ 1232.380653] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.14.auto: Registered PMU @ 0x0000200100020000 using 8 counters with Individual filter settings
[ 1232.380698] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.15.auto: option mask 0x1
[ 1232.380770] arm-smmu-v3-pmcg arm-smmu-v3-pmcg.15.auto: Registered PMU @ 0x0000200140020000 using 8 counters with Individual filter settings
```

- 使用perf list | grep pmcg可以查看系统支持的pmcg相关的时间类型:
```
Ubuntu:/ # perf list | grep pmcg
  [...]
  smmuv3_pmcg_140020/config_cache_miss/              [Kernel PMU event]
  smmuv3_pmcg_140020/config_struct_access/           [Kernel PMU event]
  smmuv3_pmcg_140020/cycles/                         [Kernel PMU event]
  smmuv3_pmcg_140020/pcie_ats_trans_passed/          [Kernel PMU event]
  smmuv3_pmcg_140020/pcie_ats_trans_rq/              [Kernel PMU event]
  smmuv3_pmcg_140020/tlb_miss/                       [Kernel PMU event]
  smmuv3_pmcg_140020/trans_table_walk_access/        [Kernel PMU event]
  smmuv3_pmcg_140020/transaction/                    [Kernel PMU event]
  [...]
```

- 使用pmcg之前需要先明确需要测试的设备是在哪个pmcg之下，pmcg的命名方式是:
  smmuv3_pmcg_<phys_addr_page>, 这里的phys_addr_page是对应SMMU PMCG基地址去掉
  低12bit。这里的设计有点不好，使用者很难找到对应的关系 :(

- 当想观测一个程序对应的SMMU上统计信息时我们可以, 比如这样:
```
  perf stat -e smmuv3_pmcg_<phys_addr_page>/tlb_miss/ <your_program>
```
  的到程序执行过程的smmu tlb miss数目。把这里的tlb_miss换成上面perf list | grep pmcg
  所示的其他事件，就可以得到其他事件的统计。

- 实际系统上可能一个smmu下接着多个外设，只想看一个外设在smmu上统计数据可以，比如
  这样:
```
  perf stat -e smmuv3_pmcg_<phys_addr_page>/tlb_miss/,filter_enable=1,filter_span=0,filter_stream_id=0x75 <your_program>
```
  上面的0x75是设备对应的stream_id，PCIe设备的话，一般就是这个设备的BDF号。
  (fix me: device function number怎么表示?)

  如果有smmu自定义的event实现，可以指定具体的event编号, 比如:
```
  perf stat -e smmuv3_pmcg_<phys_addr_page>/event=0x80/ <your_program>
```
  对于自定义的event，定义的命令是：
```
  perf stat -e smmuv4_pmcg_<phys_addr_page>/event=0x80,filter_enable=1,filter_span=0,filter_stream_id=0x75/ <your_program>
```
