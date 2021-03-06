---
title: hisi perf uncore event
tags:
  - 软件性能
  - perf
description: >-
  本文档介绍使用hisi perf uncore事件调优的方法，目前主要是perf
  uncore事件和硬件的对应关系介绍。本文基于v5.6-rc1的主线内核进行分析。
abbrlink: 86d5a04c
date: 2021-06-19 10:32:07
---

你可以使用 perf list来列出系统支持的perf事件，有一类perf事件可以用来统计CPU的
L3 cache, HHA和DDRC的事件，他们统一叫uncore事件。他们对应的内核驱动是在
linux/drivers/perf/hisilicon/*。这些uncore event的命名是这样的：
```
  hisi_sccl1_ddrc0/act_cmd/                          [Kernel PMU event]
  hisi_sccl1_ddrc0/flux_rcmd/                        [Kernel PMU event]
  [...]
  hisi_sccl1_ddrc1/act_cmd/                          [Kernel PMU event]
  hisi_sccl1_ddrc1/flux_rcmd/                        [Kernel PMU event]
  [...]
  hisi_sccl1_ddrc2/act_cmd/                          [Kernel PMU event]
  hisi_sccl1_ddrc2/flux_rcmd/                        [Kernel PMU event]
  [...]
  hisi_sccl1_ddrc3/act_cmd/                          [Kernel PMU event]
  hisi_sccl1_ddrc3/flux_rcmd/                        [Kernel PMU event]
  [...]
  hisi_sccl1_hha2/bi_num/                            [Kernel PMU event]
  hisi_sccl1_hha2/edir-hit/                          [Kernel PMU event]
  [...]
  hisi_sccl1_hha3/bi_num/                            [Kernel PMU event]
  hisi_sccl1_hha3/edir-hit/                          [Kernel PMU event]
  [...]
  hisi_sccl1_l3c10/back_invalid/                     [Kernel PMU event]
  hisi_sccl1_l3c10/prefetch_drop/                    [Kernel PMU event]
  [...]
  hisi_sccl1_l3c11/back_invalid/                     [Kernel PMU event]
  hisi_sccl1_l3c11/prefetch_drop/                    [Kernel PMU event]
  [...]
```

我们在做性能分析的时候，首先要看懂这些统计，把这些项目和程序运行的CPU对应上。
现在依次介绍相关的概念。一个完整的服务器CPU系统(我们这里不看IO)，是由物理CPU，
物理CPU中的CPU die, CPU die上的一个个CPU core组成的。一般，物理CPU支持多个互联
在一起，我们下面用chip表示一个物理CPU, 一个物理CPU里可以有多个CPU die, 在uncore
event里CPU die我们叫做sccl<n>, 这里的n是sccl的编号，其中chip0(主片)上的CPU die
分别叫sccl1和sccl3, chip1(从片)上的叫sccl5、sccl7，注意我们这里举例的系统一个
物理CPU里有两个CPU die。

一个sccl中的CPU core是四个聚集在一起成一个cluster，一般一个sccl里有6个cluster,
那么一个sccl就有24个core，一个物理CPU有48个core, 一个2P系统就有96个core。这些core
和DDR的连接如下图, 他们通过HHA和DDRC连接，DDRC和DIM条连接。sccl之间通过HHA相连。
这些core和L3 cache的连接关系(这里先不考虑L1, L2 cache)是一个sccl和一大块L3相连，
在使用上，把这一大块L3 cache分成几个partition, 一般是有几个cluster就分几个partion,
一个cluster里的core优先使用自己cluster对应的L3 partition，当然也可以使用其他的
L3 partion。

这样，我们很好看懂上面的event，比如:
```
 hisi_sccl1_l3c11/back_invalid/                     [Kernel PMU event]
```
就表示，chip0上sccl1这个CPU die上L3 partition编号是11的back_invalid事件。一般，
一个sccl对应一个NUMA node节点，一个l3c后面的编号在一个sccl上是顺序增加的。
比如，sccl1上的的各个l3c的编号是，l3c10, l3cll, l3c12, l3c13, l3c14, l3c15，那么
l3c11对应的就是这个sccl1上的core4~core7。一般，sccl1对应的就是系统里的node0,
sccl3对应node1，sccl5对应node2，sccl7对应node3。

使用numactl -H可以确定各个NUMA node里的CPU编号，比如：
```
available: 4 nodes (0-3)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
node 0 size: 0 MB
node 0 free: 0 MB
node 1 cpus: 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 48 49 50 51 52 53 54 55 56 57 58 59 60 61 62 63
node 1 size: 31912 MB
node 1 free: 30275 MB
node 2 cpus: 64 65 66 67 68 69 70 71 72 73 74 75 76 77 78 79 80 81 82 83 84 85 86 87 88 89 90 91 92 93 94 95
node 2 size: 0 MB
node 2 free: 0 MB
node 3 cpus: 96 97 98 99 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115 116 117 118 119 120 121 122 123 124 125 126 127
node 3 size: 32097 MB
node 3 free: 31514 MB
node distances:
node   0   1   2   3 
  0:  10  12  20  22 
  1:  12  10  22  24 
  2:  20  22  10  12 
  3:  22  24  12  10 
```
注意这个系统是128core的系统，无非是系统拓扑基本上不变，一个sccl里多了2个cluster。

```
                               sccl1      sccl3                 sccl5         sccl7
     DIM0 DIM1  DIM2 DIM3        /         /
       |  |       |  |          /  ...    /      chip0                               chip1
 +-----+--+-------+--+---------/---------/------------+   +-------------------------------+
 | +---+--+-------+--+--------/-+ +-----/-----------+ |   |                               |
 | | +-----+    +-----+      /  | |    /            | |   |                               |
 | | |DDRC0|    |DDRC1|     /   | |   /             | |   |                               |
 | | +---+-+    +-+---+    /    | |  /              | |   |                               |
 | |     |        |             | |                 | |   |                               |
 | |     +---+ +--+             | |                 | |   |                               |
 | |         | |                | |  ...            | |   |    ...                        |
 | |      +--+-+-+              | |                 | |   |                               |
 | |      | HHA0 +--------------+-+--               | |   |                               |
 | |      +--+---+              | |                 | |   |                               |
 | |         |         +------+ | |                 | |   |                               |
 | |   +-----+-----+   |      | | | +-----+-----+   | |   | +-----+-----+   +-----+-----+ |
 | |   |core0|core1|   |l3c<n>| | | |core0|core1|   | |   | |core0|core1|   |core0|core1| |
 | |   +-----+-----+---+      | | | +-----+-----+   | |   | +-----+-----+   +-----+-----+ |
 | |   |core2|core3|   +------+ | | |core2|core3|   | |   | |core2|core3|   |core2|core3| |
 | |   +-----+-----+   |      | | | +-----+-----+   | |   | +-----+-----+   +-----+-----+ |
 | |                   |...   | | |                 | |   |                               |
 | |   ...             |      | | | ...             | |   | ...             ...           |
 | |                   |      | | |                 | |   |                               |
 | |   +-----+-----+   |      | | | +-----+-----+   | |   | +-----+-----+   +-----+-----+ |
 | |   |core0|core1|   +------+ | | |core0|core1|   | |   | |core0|core1|   |core0|core1| |
 | |   +-----+-----+---+      | | | +-----+-----+   | |   | +-----+-----+   +-----+-----+ |
 | |   |core2|core3|   |l3c<n>| | | |core2|core3|   | |   | |core2|core3|   |core2|core3| |
 | |   +-----+-----+   |      | | | +-----+-----+   | |   | +-----+-----+   +-----+-----+ |
 | |         |         +------+ | |                 | |   |                               |
 | |      +--+---+              | |                 | |   |                               |
 | |      | HHA1 +--------------+-+--               | |   |                               |
 | |      +--+-+-+              | |  ...            | |   |    ...                        |
 | |         | |                | |                 | |   |                               |
 | |     +---+ +--+             | |                 | |   |                               |
 | |     |        |             | |                 | |   |                               |
 | | +---+-+   +--+--+          | |                 | |   |                               |
 | | |DDRC2|   |DDRC3|          | |                 | |   |                               |
 | | +-----+   +-----+          | |                 | |   |                               |
 | +--+--+-------+--+-----------+ +-----------------+ |   |                               |
 +----+--+-------+--+---------------------------------+   +-------------------------------+
      |  |       |  |               ...                     ...
    DIM0 DIM1  DIM2 DIM3
```
下面我们先看下L3 cache的各个event的含义：
```
  hisi_sccl1_l3c10/back_invalid/                     [Kernel PMU event]
  hisi_sccl1_l3c10/prefetch_drop/                    [Kernel PMU event]
  hisi_sccl1_l3c10/rd_cpipe/                         [Kernel PMU event]
  hisi_sccl1_l3c10/rd_hit_cpipe/                     [Kernel PMU event]
  hisi_sccl1_l3c10/rd_hit_spipe/                     [Kernel PMU event]
  hisi_sccl1_l3c10/rd_spipe/                         [Kernel PMU event]
  hisi_sccl1_l3c10/retry_cpu/                        [Kernel PMU event]
  hisi_sccl1_l3c10/retry_ring/                       [Kernel PMU event]
  hisi_sccl1_l3c10/victim_num/                       [Kernel PMU event]
  hisi_sccl1_l3c10/wr_cpipe/                         [Kernel PMU event]
  hisi_sccl1_l3c10/wr_hit_cpipe/                     [Kernel PMU event]
  hisi_sccl1_l3c10/wr_hit_spipe/                     [Kernel PMU event]
  hisi_sccl1_l3c10/wr_spipe/                         [Kernel PMU event]
```
  rd_cpipe, rd_spipe可以表示CPU发出的所有请求数。
  rd_hit_cpipe, rd_hit_spipe可以表示CPU发出的请求hit该L3 partition的数目。
