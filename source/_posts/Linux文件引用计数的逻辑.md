---
title: Linux文件引用计数的逻辑
tags:
  - Linux内核
  - 文件系统
description: 本文分析现在Linux内核中对打开文件引用计数的处理逻辑，目的是解答一个问题，即对设备文件的操作会不会引用到已经释放的文件上。
abbrlink: 7cc0e2f4
date: 2021-06-27 18:07:01
---

考虑这样一个场景，打开一个字符设备文件/dev/A，得到一个fd，用户态可以对这个
fd做相关的文件操作，包括ioctl, mmap, close等，内核如果保证close操作和其他
操作的同步，即不会出现close和其他文件并发执行，其他文件访问已经close掉的文件
这种情况。

内核是靠打开文件的引用计数来保证这一点的。
```
kernel/fs/open.c
filp_open
  +-> file_open_name
    +-> do_filp_open
      +-> path_openat
        +-> alloc_empty_file
	这里在创建struct file结构的时候会把里面的f_count引用计数设置为1。
```
```
kernel/fs/ioctl.c
ksys_ioctl系统调用
  +-> fdget
    +-> __fdget
      在rcu锁里得到file结构的指针
      +-> __fget_light
        +-> __fget
	  +-> get_file_rcu_many (atomic_long_add_unless(&(x)->f_count, xx, 0))
          这里只有在f_count非0的时候才会把引用计数加1。如果是0，表明已经file
	  的引用计数已经是0。__fget会去files里查fd对应的file。
  +-> fdput
    +-> fput
      +-> fput_many
        +-> atomic_long_dec_and_test(&file->f_count)
	如果减到0，在另一个内核线程中，延迟执行delay_work：
	  +-> delayed_fput_work
	    +-> delayed_fput
	      +-> __fput
	        +-> file_free(file)
```
```
kernel/fs/open.c
close系统调用
  +-> __close_fd
    +-> spin_lock(&files->file_lock)
    在锁里拿到fd对应的file结构的指针
      +-> filp_close
        +-> fput     
        如上
    +-> spin_unlock(&files->file_lock)
```
