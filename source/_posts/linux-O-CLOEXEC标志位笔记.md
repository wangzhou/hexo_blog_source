---
title: linux O_CLOEXEC标志位笔记
tags:
  - Linux内核
  - 文件系统
description: 本文介绍Linux O_CLOEXEC标志位的语意
abbrlink: 4446f88f
date: 2021-06-27 18:12:49
---

在linux系统中，open一个文件可以带上O_CLOEXEC标志位，这个表示位和用fcntl设置的
FD_CLOEXEC有同样的作用，都是在fork的子进程中用exec系列系统调用加载新的可执行
程序之前，关闭子进程中fork得到的fd。

fork之后，子进程得到父进程的完整拷贝，对于父进程已经open的文件，子进程也可以得
到一样的fd。内核里，子进程只是把fd对应的file指针指向父进程fd对应的struct file，
并且把file的引用加1。对于设置了这个标志位的fd来说，内核执行exec时，在加载二进制
可执行文件之前会调用filp_close关闭子进程的fd。可以看到filp_close的意图是关闭
子进程里的fd，但是因为父进程还持有fd的引用计数，所以这个关闭的动作只会执行诸如
文件fops对应的flush回调函数，并没有真正调用到fops的release回调函数把struct file
release掉。

这里有一个简单的测试程序：https://github.com/wangzhou/tests/tree/master/fork_exec
从test_log里可以看到，子进程执行execlp会触发设备驱动里的flush函数。

可以看到，如果这里的flush函数有对硬件的操作将有可能出错。比如这个设备正被
open在父进程里使用，子进程里调用到的flush函数也会操作到相同的struct file，可能
破坏这个正在打开的设备里的资源。
