---
title: 使用perf trace跟踪IO缺页
tags:
  - 软件调试
  - perf
description: 本文以一个实例介绍如何使用perf trace跟踪Linux内核中的trace point
abbrlink: e3e43140
date: 2021-06-19 10:29:33
---

使用perf list可以看到有trace point的软件定义的trace点。(fix me: 要开什么内核选项)
这些软件定义的trace point点要在代码里提前预埋，执行程序的时候可以用perf trace
把需要的信息统计出来。

我们拿Linux内核里IOMMU统计IO缺页的event做例子看看。这个event的定义在:
include/trace/events/iommu.h里：
```
TRACE_EVENT(dev_fault,

	TP_PROTO(struct device *dev,  struct iommu_fault *evt),

	TP_ARGS(dev, evt),

	TP_STRUCT__entry(
		__string(device, dev_name(dev))
		__field(int, type)
		__field(int, reason)
		__field(u64, addr)
		__field(u64, fetch_addr)
		__field(u32, pasid)
		__field(u32, grpid)
		__field(u32, flags)
		__field(u32, prot)
	),

	TP_fast_assign(
		__assign_str(device, dev_name(dev));
		__entry->type = evt->type;
		if (evt->type == IOMMU_FAULT_DMA_UNRECOV) {
			__entry->reason		= evt->event.reason;
			__entry->flags		= evt->event.flags;
			__entry->pasid		= evt->event.pasid;
			__entry->grpid		= 0;
			__entry->prot		= evt->event.perm;
			__entry->addr		= evt->event.addr;
			__entry->fetch_addr	= evt->event.fetch_addr;
		} else {
			__entry->reason		= 0;
			__entry->flags		= evt->prm.flags;
			__entry->pasid		= evt->prm.pasid;
			__entry->grpid		= evt->prm.grpid;
			__entry->prot		= evt->prm.perm;
			__entry->addr		= evt->prm.addr;
			__entry->fetch_addr	= 0;
		}
	),

	TP_printk("IOMMU:%s type=%d reason=%d addr=0x%016llx fetch=0x%016llx pasid=%d group=%d flags=%x prot=%d",
		__get_str(device),
		__entry->type,
		__entry->reason,
		__entry->addr,
		__entry->fetch_addr,
		__entry->pasid,
		__entry->grpid,
		__entry->flags,
		__entry->prot
	)
);
```
在需要打点的地方插入一个trace_dev_fault(dev, evt)就好，其中dev是TP_PROTO里定义的
struct device *dev, evt是里面定义的struct iommu_fault *evt。

TP_STRUCT__entry定义记录结构里各个域段的定义。TP_fast_assign定义域段记录的值。
TP_printk定义打印的方式。

以UADK里一个测试用力为例，我们看看怎么用perf trace收集IO page fault的信息。具体
的运行命令如下:
```
sudo ./perf trace -o log_sva -a -e iommu:* numactl --cpubind 1 --membind 1  \
test_hisi_sec --perf --async --pktlen 1024 --block 8192 --blknum 100000 \
--times 1000000 --multi 1 --ctxnum 1
```
-o后面加需要存放log的文件。注意, 需要sudo权限，需要-a，不然无法看到
iommu:dev_fault的事件，另外这个用力要使用block 8192才会观察到iommu:dev_fault事件

观察到的log_sva里的记录可能是这样的：
```
     0.000 :0/0 iommu:unmap:IOMMU: iova=0x00000000fdb67000 size=4096 unmapped_size=4096
     0.030 :0/0 iommu:unmap:IOMMU: iova=0x00000000fdabe000 size=4096 unmapped_size=4096
     0.396 test_hisi_sec/115486 iommu:map:IOMMU: iova=0x00000000fbfe9000 paddr=0x000000217e0c8000 size=4096
     0.432 test_hisi_sec/115486 iommu:unmap:IOMMU: iova=0x00000000fbfe9000 size=4096 unmapped_size=4096
     0.444 test_hisi_sec/115486 iommu:map:IOMMU: iova=0x00000000fbfe9000 paddr=0x000000217e0c8000 size=4096
     0.465 test_hisi_sec/115486 iommu:unmap:IOMMU: iova=0x00000000fbfe9000 size=4096 unmapped_size=4096
   671.920 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x000000002321c000 fetch=0x0000000000000000 pasid=1 group=138 flags=3 prot=1
   671.961 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000023220000 fetch=0x0000000000000000 pasid=1 group=119 flags=3 prot=1
   671.983 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000023230000 fetch=0x0000000000000000 pasid=1 group=158 flags=3 prot=1
   672.003 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000023234000 fetch=0x0000000000000000 pasid=1 group=132 flags=3 prot=1
   672.024 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x000000002323c000 fetch=0x0000000000000000 pasid=1 group=135 flags=3 prot=1
   672.041 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000023232000 fetch=0x0000000000000000 pasid=1 group=120 flags=3 prot=2

  [...]

  1946.610 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000084d82000 fetch=0x0000000000000000 pasid=1 group=122 flags=3 prot=2
  1946.636 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000084da6300 fetch=0x0000000000000000 pasid=1 group=88 flags=3 prot=2
  1946.659 irq/33-arm-smm/873 iommu:dev_fault:IOMMU:0000:76:00.0 type=2 reason=0 addr=0x0000000084d8a180 fetch=0x0000000000000000 pasid=1 group=86 flags=3 prot=2
  3031.527 :0/0 iommu:unmap:IOMMU: iova=0x00000000fdbe2000 size=4096 unmapped_size=4096
  3031.550 :0/0 iommu:unmap:IOMMU: iova=0x00000000fdb68000 size=4096 unmapped_size=4096
  3031.499 test_hisi_sec/115486 iommu:map:IOMMU: iova=0x00000000fbfe9000 paddr=0x000000217e0c8000 size=4096
  3031.557 test_hisi_sec/115486 iommu:unmap:IOMMU: iova=0x00000000fbfe9000 size=4096 unmapped_size=4096
```

除了用perf trace跟踪，也可以用ftrace跟踪。这需要ftrace的目录下(一般在/sys/kernel/debug/tracing)
的event里使能对应的trace point点，这样再去trace就可以看到输出的打印。

也可以在需要跟踪的地方简单的加一个trace_printk()的打印，把对应的模块写到
set_ftrace_filter: echo ':mod:xxx_module_name' > set_ftrace_filter。然后再去
trace，也可以看到输出的打印。
