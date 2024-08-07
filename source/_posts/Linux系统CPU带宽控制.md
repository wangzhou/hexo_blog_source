---
title: Linux系统CPU带宽控制
tags:
  - Linux内核
  - 调度
  - cgroup
description: >-
  本文总结Linux内核里关于CPU使用带宽控制的基本逻辑。Linux是一个多用户的操作
  系统，管理员可以通过一定的手段控制用户(线程)对CPU资源的使用，本文分析内核 对这个特性的支持逻辑。
abbrlink: 6239
date: 2024-03-15 19:12:02
categories:
---

使用cgroup限制线程对CPU使用
----------------------------

利用CPU的cgroup sysfs接口，在/sys/fs/cgroup/cpu下创建测试目录：
```
root@m1:/sys/fs/cgroup/cpu# pwd
/sys/fs/cgroup/cpu
root@m1:/sys/fs/cgroup/cpu# mkdir my_test
root@m1:/sys/fs/cgroup/cpu# cd my_test/
root@m1:/sys/fs/cgroup/cpu/my_test# ls
cgroup.clone_children  cpuacct.usage_all          cpuacct.usage_sys   cpu.shares      notify_on_release
cgroup.procs           cpuacct.usage_percpu       cpuacct.usage_user  cpu.stat        tasks
cpuacct.stat           cpuacct.usage_percpu_sys   cpu.cfs_period_us   cpu.uclamp.max
cpuacct.usage          cpuacct.usage_percpu_user  cpu.cfs_quota_us    cpu.uclamp.min
root@m1:/sys/fs/cgroup/cpu/my_test# cat cpu.cfs_period_us 
100000
root@m1:/sys/fs/cgroup/cpu/my_test# cat cpu.cfs_quota_us 
-1
```

写一个死循环的demo(PID是3818875)，htop查看这个demo跑起来后，对应的CPU占用率为100%：
```
  1  [                                            0.0%]   Tasks: 138, 464 thr; 2 running
  2  [|                                           0.7%]   Load average: 0.97 0.45 0.22 
  3  [|                                           0.7%]   Uptime: 7 days, 10:12:34
  4  [||||||||||||||||||||||||||||||||||||||||||100.0%]
  Mem[|||||||||||||||||||||||||||||||||||||1.23G/7.75G]
  Swp[|||                                   111M/2.00G]

    PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
3818875 sherlock   20   0  1788   432   368 R 99.7  0.0  1:59.92 ./a.out
```

把PID写入测试目录的tasks文件，并把cpu.cfs_period_us一半的数值写入到cpu.cfs_quota_us，
这个表示在cpu.cfs_period_us的时间内只容许对应线程运行cpu.cfs_quota_us时间。观察
htop发现，demon死循环线程的CPU占用率大概将为50%。
```
root@m1:/sys/fs/cgroup/cpu/my_test# echo 3818875 > tasks 
root@m1:/sys/fs/cgroup/cpu/my_test# echo 50000 > cpu.cfs_quota_us 
```
```
  1  [                                            0.0%]   Tasks: 138, 464 thr; 2 running
  2  [|                                           0.7%]   Load average: 0.75 0.54 0.28 
  3  [|                                           0.7%]   Uptime: 7 days, 10:14:10
  4  [|||||||||||||||||||||||||                  51.0%]
  Mem[|||||||||||||||||||||||||||||||||||||1.23G/7.75G]
  Swp[|||                                   111M/2.00G]

    PID USER      PRI  NI  VIRT   RES   SHR S CPU% MEM%   TIME+  Command
3818875 sherlock   20   0  1788   432   368 R 49.5  0.0  3:21.22 ./a.out
```

基本逻辑
---------

todo: ...

代码分析
---------

todo: ...
