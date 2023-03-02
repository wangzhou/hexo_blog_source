---
title: 使用eBPF得到内核执行过程的时间分布
tags:
  - 软件性能
description: >-
  本文来自于一个这样的需求，我们需要统计Linux内核里一段执行流程执行时间的 分布情况，得到这个信息对于我们做系统优化有很大的帮助。除了帮助我们得到系统
  运行情况的统计特征，我们还可以基于这样的基准测试来衡量我们的优化效果。本 文以一个实际的例子介绍怎么得到这样的统计信息。
abbrlink: 12b4e640
date: 2021-06-18 23:53:18
---

具体例子介绍
---------------

 在SVA的使用场景中，IO缺页比较影响系统的性能。为此，我们需要观测在一个段程序执行
 的时候，系统中发生IO缺页的次数, 以及IO缺页的统计特征，比如IO缺页的平均时间、方差
 和分布情况。我们可以在内核代码中加tracepoint点，然后用perf或者是ftrace得到trace
 信息，然后用脚本处理得到上面的信息。但是，本文介绍的是用eBPF的方法得到这样的
 信息。

 使用eBPF的方法，可以在把一段用户态写的代码放到内核执行，这段用户态写的代码可以
 在已有的内核tracepoint点上附着。就是每次内核里跑到tracepoint点的时候，都可以跑
 一下用户态写的代码，用户可以自由编程统计自己想要的信息。

 eBPF在内核中提供里各种help函数，方便统计代码调用。同时现在也有关于eBPF的用户态
 库(e.g. <https://github.com/iovisor/bcc.git>)，方便在用户态编写代码，记录和处理
 得到的信息。

内核配置
-----------
 
 我们使用<https://github.com/Linaro/linux-kernel-uadk> branch: uacce-devel-5.11
 的分支测试，需要打上如下第3节中的内核补丁。

 如下是需要打开的内核编译选项:

 make ARCH=arm64 defconfig
 CONFIG_UACCE=y
 CONFIG_DEV_HISI_ZIP=y
 CONFIG_DEV_HISI_QM=y
 CONFIG_ARM_SMMU_V3=y (默认)
 CONFIG_ARM_SMMU_V3_SVA=y (默认)
 CONFIG_IOMMU_SVA_LIB=y (默认)
 CONFIG_PCI_PASID=y (默认)
 CONFIG_FTRACE=y
 (以上是业务相关的)

 CONFIG_BPF_SYSCALL=y
 CONFIG_BPF_KPROBE_OVERRIDE=y
 CONFIG_KPROBE=y

 CONFIG_SQUASHFS_XZ=y
 CONFIG_CGROUP_FREEZER=y
 (以上是ebpf相关的)

用户态环境搭建
-----------------

 本文测试所使用的环境是Ubuntu 16.04, 这个版本的ubuntu没有bcc相关的仓库，外部仓库
 现在也无法使用了(<http://www.brendangregg.com/blog/2016-06-14/ubuntu-xenial-bcc-bpf.html>)。

 所以本文测试使用了snap来安装bcc相关的环境，snap是ubuntu apt之外的另一个包管理器。
```
	sudo apt-get install snap
	sudo snap install bcc
```
 如上会把snap管理下的bcc包安装到/snap/*下。 ls -al /snap
```
total 28
drwxr-xr-x  6 root root 4096 2月  20 02:34 .
drwxr-xr-x 22 root root 4096 10月  1 15:48 ..
drwxr-xr-x  3 root root 4096 2月  20 02:34 bcc
drwxr-xr-x  2 root root 4096 2月  20 05:16 bin
drwxr-xr-x  3 root root 4096 2月  20 02:33 core18
-r--r--r--  1 root root  548 10月 17  2019 README
drwxr-xr-x  3 root root 4096 2月  20 02:32 snapd
```

为python增加搜索库的路径，可以看到snap管理的bcc python库都在snap自己的路劲下。
```
PYTHONPATH="/snap/bcc/146/usr/lib/python3/dist-packages:{$PYTHONPATH}"
export PYTHONPATH

LD_LIBRARY_PATH="/snap/bcc/146/usr/lib/aarch64-linux-gnu/:{$LD_LIBRARY_PATH}"
export LD_LIBRARY_PATH
```

注意，这里编译bcc代码的时候，会使用到内核头文件。本文尝试了各种内核头文件
导出的方式都还是一直有内核头文件找不到。索性把查找头文件的目录又重新软连接到
使用内核的源码路径上。为了简单起见，本文的测试使用了本地编译内核的方式：
```
	ln -s <your kernel source> /lib/modules/<your_kernel_magic>/build
```

代码示例
-----------
内核补丁:
```
diff --git a/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c b/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c
index 8d29aa1be282..637e95c237a1 100644
--- a/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c
+++ b/drivers/iommu/arm/arm-smmu-v3/arm-smmu-v3.c
@@ -32,6 +32,8 @@
 
 #include <linux/amba/bus.h>
 
+#include <trace/events/smmu.h>
+
 #include "arm-smmu-v3.h"
 #include "../../iommu-sva-lib.h"
 
@@ -945,6 +947,8 @@ static int arm_smmu_page_response(struct device *dev,
 	 * forget.
 	 */
 
+	trace_io_fault_exit(dev, resp->pasid);
+
 	return 0;
 }
 
@@ -1474,6 +1478,8 @@ static int arm_smmu_handle_evt(struct arm_smmu_device *smmu, u64 *evt)
 	struct iommu_fault_event fault_evt = { };
 	struct iommu_fault *flt = &fault_evt.fault;
 
+	trace_io_fault_entry(smmu->dev, FIELD_GET(EVTQ_0_SSID, evt[0]));
+
 	/* Stage-2 is always pinned at the moment */
 	if (evt[1] & EVTQ_1_S2)
 		return -EFAULT;
diff --git a/include/trace/events/smmu.h b/include/trace/events/smmu.h
index e9b648407102..4d96bfd20726 100644
--- a/include/trace/events/smmu.h
+++ b/include/trace/events/smmu.h
@@ -82,6 +82,28 @@ DEFINE_EVENT(smmu_mn, smmu_mn_free, TP_PROTO(unsigned int pasid), TP_ARGS(pasid)
 DEFINE_EVENT(smmu_mn, smmu_mn_get, TP_PROTO(unsigned int pasid), TP_ARGS(pasid));
 DEFINE_EVENT(smmu_mn, smmu_mn_put, TP_PROTO(unsigned int pasid), TP_ARGS(pasid));
 
+DECLARE_EVENT_CLASS(smmu_io_fault_class,
+	TP_PROTO(struct device *dev, unsigned int pasid),
+	TP_ARGS(dev, pasid),
+
+	TP_STRUCT__entry(
+		__string(dev, dev_name(dev))
+		__field(int, pasid)
+	),
+	TP_fast_assign(
+		__assign_str(dev, dev_name(dev));
+		__entry->pasid = pasid;
+	),
+	TP_printk("dev=%s pasid=%d", __get_str(dev), __entry->pasid)
+);
+
+#define DEFINE_IO_FAULT_EVENT(name)       \
+DEFINE_EVENT(smmu_io_fault_class, name,        \
+	TP_PROTO(struct device *dev, unsigned int pasid), \
+	TP_ARGS(dev, pasid))
+
+DEFINE_IO_FAULT_EVENT(io_fault_entry);
+DEFINE_IO_FAULT_EVENT(io_fault_exit);
 
 #endif /* _TRACE_SMMU_H */

```
如上，为了在eBPF的跟踪代码里跟踪io page fault的流程，我们在SMMU io page fault
的入口和出口出增加了新的tracepoint点。增加trancepoint的方法还可以参考[这里](https://wangzhou.github.io/使用perf-trace跟踪IO缺页/)。

用户态python脚本, ebpf_smmu_iopf.py。可以参考bcc里自带的tools, 看看怎么写这些
脚本: <https://github.com/iovisor/bcc.git>
```
#!/usr/bin/python3
#
from __future__ import print_function
from bcc import BPF
from ctypes import c_ushort, c_int, c_ulonglong
from time import sleep
from sys import argv

def usage():
        print("USAGE: %s [interval [count]]" % argv[0])
        exit()

# arguments
interval = 5
count = -1
if len(argv) > 1:
        try:
                interval = int(argv[1])
                if interval == 0:
                        raise
                if len(argv) > 2:
                        count = int(argv[2])
        except: # also catches -h, --help
                usage()

# load BPF program
b = BPF(text="""
#include <uapi/linux/ptrace.h>

BPF_HASH(start, u64);
BPF_HISTOGRAM(dist);

TRACEPOINT_PROBE(smmu, io_fault_entry)
{
        u64 ts;

        ts = bpf_ktime_get_ns();
        start.update((unsigned int *)args->pasid, &ts);

        return 0;
}

TRACEPOINT_PROBE(smmu, io_fault_exit)
{
        u64 *tsp, delta;
        u64 pasid;

        tsp = start.lookup((unsigned int *)args->pasid);

        if (tsp != 0) {
                delta = bpf_ktime_get_ns() - *tsp;
                dist.increment(bpf_log2l(delta));
                start.delete((unsigned int *)args->pasid);
        }

        return 0;
}
""")

# header
print("Tracing... Hit Ctrl-C to end.")

# output
loop = 0
do_exit = 0
while (1):
        if count > 0:
                loop += 1
                if loop > count:
                        exit()
        try:
                sleep(interval)
        except KeyboardInterrupt:
                pass; do_exit = 1

        print()
        b["dist"].print_log2_hist("nsecs")
        b["dist"].clear()
        if do_exit:
                exit()
```

运行效果
-----------
```
 sudo ./ebpf_smmu_iopf.py

 另一个窗口运行：
 sudo zip_perf_test -b 8192 -s 819200000
```
得到如下的iopf时延的分布图:
```
[...]
```

在过程中测试eBPF的环境是否搭建ok，也可以跑下/snap/bin自带的程序，比如跑bcc.cpudist
我们得到了这样的分布图：
```
Tracing on-CPU time... Hit Ctrl-C to end.

     usecs               : count     distribution
         0 -> 1          : 193      |*************************               |
         2 -> 3          : 49       |******                                  |
         4 -> 7          : 35       |****                                    |
         8 -> 15         : 41       |*****                                   |
        16 -> 31         : 37       |****                                    |
        32 -> 63         : 116      |***************                         |
        64 -> 127        : 14       |*                                       |
       128 -> 255        : 2        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 3        |                                        |
      1024 -> 2047       : 12       |*                                       |
      2048 -> 4095       : 61       |********                                |
      4096 -> 8191       : 96       |************                            |
      8192 -> 16383      : 97       |************                            |
     16384 -> 32767      : 162      |*********************                   |
     32768 -> 65535      : 301      |****************************************|
```
