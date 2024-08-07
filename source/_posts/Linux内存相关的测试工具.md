---
title: Linux内存相关的测试工具
tags:
  - 内存管理
description: 本文用来整理Linux内存相关的测试工具
abbrlink: 43af8eb9
date: 2021-06-19 10:11:01
---

smem
-------
```
  PID User     Command                         Swap      USS      PSS      RSS 
40200 test     bash                               0   524.0K   687.0K     3.1M 
39757 test     bash                               0   524.0K   692.0K     3.2M 
40203 test     /usr/bin/python /usr/bin/sm        0     6.1M     6.2M     7.7M 
```
运行smem -k命令可以看到系统里进程占用内存的情况。如上各个SS的含义是:

 USS: unique set size. 表示的是进程独占的物理内存，不包含动态库占的内存。
 PSS: proportional set size. 进程自己的物理内存 + 动态库折算到进程里的内存。
 RSS: resident set size. 进程实际使用的物理内存，包括共享库内存。
 VSS: virtual set size. 进程使用的所有虚拟内存，包括共享库虚拟内存。

/proc/buddyinfo
------------------
```
Node 1, zone      DMA     10     12     11     11     13     11     10     10      6      6    226 
Node 1, zone    DMA32      1      0      1      1      4      2      4      3      4      4    176 
Node 1, zone   Normal  15807  52091  12723    877    258    703    579    194     35     34   5546 
Node 3, zone   Normal  54266  41801  23819  10038   3476    641     65     29     13     19   6714 
```
表示系统里伙伴系统中，2^1, 2^2...连续页有多少个，这个可以看到伙伴系统内存碎片的
情况。

/sys/kernel/mm/*
-------------------
```
├── hugepages
├── ksm
├── swap
└── transparent_hugepage
```
可以看到如上各个特性的一些统计值。

pidof + pmap + /proc/pid/maps /proc/pid/smaps /proc/pid/stack
----------------------------------------------------------------
./a.out &
pmap -x `pidof a.out`
```
41677:   ./a.out
Address           Kbytes     RSS   Dirty Mode  Mapping
0000000000400000       4       4       0 r-x-- a.out
0000000000400000       0       0       0 r-x-- a.out
0000000000410000       4       4       4 r---- a.out
0000000000410000       0       0       0 r---- a.out
0000000000411000       4       4       4 rw--- a.out
0000000000411000       0       0       0 rw--- a.out
0000ffff8dac8000    1204     480       0 r-x-- libc-2.23.so
0000ffff8dac8000       0       0       0 r-x-- libc-2.23.so
0000ffff8dbf5000      60       0       0 ----- libc-2.23.so
0000ffff8dbf5000       0       0       0 ----- libc-2.23.so
0000ffff8dc04000      16      16      16 r---- libc-2.23.so
0000ffff8dc04000       0       0       0 r---- libc-2.23.so
0000ffff8dc08000       8       8       8 rw--- libc-2.23.so
0000ffff8dc08000       0       0       0 rw--- libc-2.23.so
0000ffff8dc0a000      16       8       8 rw---   [ anon ]
0000ffff8dc0a000       0       0       0 rw---   [ anon ]
0000ffff8dc0e000     112     112       0 r-x-- ld-2.23.so
0000ffff8dc0e000       0       0       0 r-x-- ld-2.23.so
0000ffff8dc2b000       8       8       8 rw---   [ anon ]
0000ffff8dc2b000       0       0       0 rw---   [ anon ]
0000ffff8dc37000       8       0       0 r----   [ anon ]
0000ffff8dc37000       0       0       0 r----   [ anon ]
0000ffff8dc39000       4       4       0 r-x--   [ anon ]
0000ffff8dc39000       0       0       0 r-x--   [ anon ]
0000ffff8dc3a000       4       4       4 r---- ld-2.23.so
0000ffff8dc3a000       0       0       0 r---- ld-2.23.so
0000ffff8dc3b000       8       8       8 rw--- ld-2.23.so
0000ffff8dc3b000       0       0       0 rw--- ld-2.23.so
0000ffffc48f9000     132      12      12 rw---   [ stack ]
0000ffffc48f9000       0       0       0 rw---   [ stack ]
---------------- ------- ------- ------- 
total kB            1592     672      72
```

/proc/pid/smaps反应一个进程里的各个vma里的信息。提供的信息比/proc/pid/maps多

numastat -p <pid>
--------------------
```
Per-node process memory usage (in MBs) for PID 417 (a.out)
                           Node 0          Node 1           Total
                  --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.01            0.00            0.01
Stack                        0.00            0.00            0.01
Private                      0.17            0.40            0.57
----------------  --------------- --------------- ---------------
Total                        0.18            0.41            0.59
```
可以看到一个进程在各个numa节点上的内存使用情况，可以用这个命令观察一段时间进程
的内存在各个numa之间使用的情况。

mallopt
----------
C库中设定内存分配配置的接口, 通过mallopt这个函数可以改变malloc/free的行为。

mbind/madvise
----------------
mbind是对应系统调用的封装，可以把虚拟内存对应的物理内存和对应的numa节点绑定。
madvise也是对应系统调用的封装，可以从用户态传递一些内存相关的属性给内核。

getrusage
------------
getrusage是一个库函数，可以得到一个进程的相关资源的使用情况。其中包括内存使用
的统计。比如其中，ru_minflt就是系统里初次分配内存这种缺页的次数，而ru_majflt
是从swap分区里换入内存这样缺页的数目。

proc下的关于内存的信息
-------------------------
/proc/slabinfo
/proc/iomem
/proc/meminfo
/proc/vmstat
/proc/zoneinfo
/proc/vmallocinfo
/proc/sys/vm/*
