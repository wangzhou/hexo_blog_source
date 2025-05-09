---
title: Linux initramfs分析
abbrlink: 39913
date: 2022-11-10 20:24:44
tags: [Linux内核, 文件系统]
description: "本文分析Linux内核里initramfs的代码逻辑。分析基于的内核版本是v6.0，qemu版本是v7.1.50。"
categories:
---

代码逻辑
=========

```
 /* 从内核的c代码入口看起 */
 start_kernel
       /* 这个函数已经接近start_kernel末尾，但是内容却无比丰富 */
   +-> arch_call_rest_init
         /* 在这个函数里拉起1号进程和2号线程 */
     +-> rest_init
           /*
            * kernel_init是1号进程对应的函数，这个函数有点特殊，这个函数除了最终
            * 拉起第一个用户态进程外，还做了一堆初始化操作。
            */
       |
     --+-> user_mode_thread(kernel_init, NULL, CLONE_FS)
       |
         +-> kernel_init
               /* 进来啥也不干，先等2号线程起来 */
               wait_for_complete(...)
               /*
                * 继续初始化一堆东西: smp, smp schedule, work queue, 各种驱动的初始化,
                * 各种__init函数的调用，跑内核自带的各种测试，等待initramfs解压完成，
                * 其中最后一个和本文的主题相关，在这之前的do_basic_setup()里会开始
                * 做initramfs/initrd的解压。
                */
           +-> kernel_init_freeable
             +-> do_basic_setup
                   /*
                    * 这是一个rootfs_initcall修饰过的函数，在do_basic_setup依次调用
                    * 各种类型的__init函数的时候，会调用到。这个函数的本质是把do_populate_rootfs
                    * 放到一个workqueue里去执行。
                    *
                    * 对于rootfs.cpio.gz的压缩过的内存文件系统，这个函数里会做相关
                    * 的解压缩操作。
                    */
               +-> populate_rootfs
                 +-> async_schedule_domain(do_populate_rootfs, ...)
             +-> wait_for_initramfs

       |   /* 创建threadd这个内核线程，这个线程是所有内核线程的父线程，他就是2号线程 */
     --+-> kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES)
       |

       +-> cpu_startup_entry()
```
 总结下如上的线程模型，内核执行到这里会拉起1号和2号进程，这个时候会出现3个进程并发
 的场景，这三个进程分别是0/1/2号进程。kernel_init会等2号线程启动后才会继续跑。可见
 initramfs的解压缩是被放到内核线程里去跑的，1号线程会异步的等initramfs解压缩完后
 再继续跑。

 1号线程等解压完成在wait_for_initramfs里，这个函数用了wait_event，这个函数wait
 在D状态(TASK_UNINTERRUPTIBLE)，这个状态的线程只能被特定事件唤醒，内核里会定时检查
 D状态的线程，如果超过一定事件没有被唤醒(一般是120s)，就会报异常。所以，如果解压
 过程超过120s，会导致等待时间超过120s，进而报异常(linux/kernel/hung_task.c)。
