---
title: Linux内核writeback逻辑分析
tags:
  - Linux内核
  - 内存管理
  - 文件系统
  - Block子系统
description: 本文分析Linux内核的writeback的基本逻辑，分析依赖的内核版本是v6.8-rc5。
abbrlink: 47389
date: 2024-06-09 17:33:40
categories:
---

## 基本逻辑

一般情况下Linux的write系统调用是把数据写到page cache，随后会有专门的内核线程把
page cache里的脏数据写回到持久存储设备上。当脏数据超过一定的比例时，write系统调
用会同步等待一定的脏数据回写完成。

Linux也提供了强制回刷脏数据的系统调用(它们的代码在kernel/fs/sync.c)，比如，sync()
把系统上所有分区上的脏数据写回硬盘(后面我们不区分ssd/hhd/nvme等不同的存储介质，
统一都称硬盘)，syncfs(fd)同步回刷fd所在的分区上的文件系统上的脏数据。

Linux通过proc文件系统提供了writeback相关的配置接口，proc/sysfs提供了对应的统计参
数对外展示接口。

## 数据结构

struct super_block *sb: 表示一个文件系统的实例。
struct backing_dev_info *bdi: 表示一个文件系统实例对应的回写设备, sb->s_bdi是其引用。
struct bdi_writeback *wb: 表示bdi上和writeback相关的数据，bdi上的wb_list收集所有
                          bdi_writeback。为啥bdi上会有多个bdi_writeback?
struct wb_writeback_work *work: 表示一个回写任务，挂在bdi_writeback的work_list。

这里bdi/wb等都和block设备有关系，实际上bdi/wb的每个实例的初始化也和block设备有关
系，其初始化的调用路径是：
```
/* linux/block/genhd.c */
__alloc_disk_node
      /* linux/mm/backing-dev.c */
  +-> bdi_alloc
    +-> bdi_init
          /* 根据是否支持cgroup writeback与否，有两个不同的版本 */
      +-> cgwb_bdi_init
            /*
             * 初始化bdi里的wb，回写相关的数据结构，参数配置，回写线程函数在这里
             * 初始化。其中就包括：wb->b_dirty/b_io/b_more_io等，wb->dwork的wb_workfn,
             * wb_workfn为回写workqueue上执行的函数。
             */
        +-> wb_init
```

文件的inode会和page cache的描述address_space联系到一起，也会和文件的回写设备wb联
系到一起。这个联系是在fs/fs-writeback.c的__inode_attach_wb中建立的。

## 代码分析

### write系统调用

先看write系统调用的基本逻辑，基本的调用逻辑如下：
```
/* kernel/fs/read_write.c */
ksys_write->vfs_write->new_sync_write->call_write_iter->ext4_file_write_iter
->ext4_buffered_write_iter
```

如上逻辑，在ext4_buffered_write_iter中的generic_perform_write写page cache，并且
标记脏数据。其中的逻辑有：(以ext4为例说明)
```
generic_perform_write
  +-> ext4_write_begin
  +-> 更新数据
  +-> ext4_write_end
    +-> block_write_end
      +-> __block_commit_write
        +-> -> mark_buffer_dirty
          +-> __mark_inode_dirty
            +-> sb->s_op->dirty_inode /* 挂到哪个链表上了? */
  +-> balance_dirty_pages_ratelimited
```
标记脏页的逻辑在ext4_write_end里，其中会标记inode、对应页面为脏页、相关脏页相关
的统计数据等等。

ext4_write_end之后有balance_dirty_pages_ratelimited，这个函数里会检测脏页的比例
是否超过一定比例，如果超过一定的比例，启动触发会写，并同步等待。(注意这里怎么同步
等待?)

### syncfs等系统调用

syncfs的基本逻辑如下：
```
/* kernel/sync.c */
syncfs
  +-> sync_filesystem
    +-> sync_inodes_sb
      +-> bdi_split_work_to_wbs
        +-> wb_queue_work(wb, work)
```
syncfs(fd)把fd所在分区上的文件系统上的脏页写回硬盘，syncfs所在的线程把回写任务交
给内核writeback线程，然后同步等待，直到脏页回写完成解开syncfs的等待。

### writeback内核线程

writeback线程的名字就是writeback，相关的代码路径在mm/backing-dev.c，内核初始化的
时候创建一个名字为writeback的unbound workqueue, default_bdi_init->alloc_workqueue。
创建的workqueue为bdi_wq，所以，系统里其它地方的回写任务会都发到bdi_wq。

syncfs的分析中可以看出，回写任务最后通过wb_queue_work放到writeback的workqueue上
运行，workqueue执行的函数是wb_workfn。其中的具体逻辑是：
```
/* fs/fs-writeback.c */
wb_workfn
  +->wb_do_writeback
    +-> wb_writeback
      +-> writeback_sb_inodes。
        +-> __writeback_single_inode
          +-> do_writepages
            +-> mapping->a_ops->writepages(ext4_writepages)
```

对于syncfs，这里调用的是writeback_sb_inodes。注意，writeback_sb_inodes里，从
bdi_writeback->b_io里取出每个要会写的脏inode，依次执行__writeback_single_inode。
所以，这里要看write什么时候把脏页加到这个b_io上。

另外，__writeback_single_inode会更新inode和实际的脏。do_writepages还在vfs的writeback
逻辑里，writepages已经到对应的文件系统回调函数里。注意这里ext4在处理了自己的逻辑后，
把请求转成bio，下发block层。

具体文件系统中的处理逻辑，这里以ext4为例。这里的submit_bio已经进入block层。
```
ext4_writepages
  +->ext4_do_writepages
    +->ext4_io_submi
         /* linux/block/blk-core.c */
      +->submit_bio。
```

除了像sync这样由用户直接触发page cache回写，内核还有定时的回写任务。内核在wb_workfn
以及inode标脏(__mark_inode_dirty)的时候，通过一个delay workqueue任务，唤醒writeback
线程工作，其中delay时间是可以配置的。
```
wb_workfn/__mark_inode_dirty
  +->wb_wakeup_delayed
        /*
         * timeout为dirty_writeback_internal通过一定的单位变化得到，这个值对应的
         * 的接口是/proc/sys/vm/dirty_writeback_centisecs，单位是10ms，也就是proc
         * 文件系统中的值除以100的秒数值，默认为5s。
         */
    +-> queue_delay_work(bdi_wq, &wb->dwork, timeout)
```

### block层

```
submit_bio
  +->submit_bio_noacct
    +->submit_bio_noacct_nocheck
      +->__submit_bio_noacct_mq
        +->__submit_bio
             /* 这里进入block层的多队列处理 */
          +->blk_mq_submit_bio
```
注意，这里要搞清楚bio进来的请求是怎么缓存的？看起来是有个current->bio_list。如果
有current->bio_list，那么把请求挂在bio_list就返回了。

## writeback相关的观测和控制手段

/proc/sys/vm目录有和writeback相关的配置，除了如上的dirty_writeback_centisecs还有：
dirty_backgroud_bytes表示周期性的回写线程在脏页超过对应的bytes时启动回写，默认是0，
dirty_backgroud_ratio表示周期性的回写线程在脏页超过对应的比例时启动回写，默认是
10%，dirty_bytes/dirty_ratio表示write系统调用在脏页超过一定的bytes或者比例时启动
回写，向drop_caches写1，释放相应的page cache。

Linux内核在文件系统、writeback逻辑、block层里添加了大量的tracepoint，可以根据这些
tracepoint点观测系统的运行情况。基于tracepoint封装的block层观测工具有blktrace系列
(blktrace/blkparse/btt)工具等。

对于blktrace，我们可以把trace点和btt输出中的各个点对应起来，其中trace_block_getrq
btt统计数据中G那个点，submit_bio_noacct_nocheck中的trace_block_bio_queue是Q点。
所以，Q2G的延时比较大，就是bio在bio_list堵了很久。

writeback:global_dirty_state可以得到和脏页相关的统计数据。可以使用如下命令得到相
关统计log：perf trace -a -e writeback:global_dirty_state -o log --time

内核在运行时，打点记录了相关writeback统计信息，并通过proc和sysfs导出了这些信息。
相关的接口有/proc/vmstat、/proc/meminfo、/sys/kernel/debug/bdi/x:x/stats。Linux
的一些工具基于如上接口封装出了使用更加方便的工具，比如sar、vmstat、iostat等。
