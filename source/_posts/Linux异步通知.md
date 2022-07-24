---
title: Linux异步通知
tags:
  - Linux内核
description: 本文是Linux异步通知的一个学习笔记，读者可以参看此文快速获得相关的知识。
abbrlink: 60c51bec
date: 2021-06-28 23:55:17
categories:
---

Linux系统中有很多内核和用户态程序通知的机制，比如event fd, netlink和异步通知。
通过这些机制内核可以主动给用户态程序发送消息。本文讨论异步通知的基本用法。

利用异步通知机制可以实现从内核中向设备文件绑定的进程发送特定信号。把异步通知用
起来需要内核中设备驱动和用户态程序的配合。在这里有一个实例程序可以直接下载，
然后在虚拟机里运行: https://github.com/wangzhou/tests/tree/master/fasync_test

如这个示例中的驱动代码，为了支持异步通知，我们需要实现设备文件操作中的.fasync
回调，实现的方式也很简单，就是调用下标准的fasync_helper函数向内核注册一个
fasync_struct, 其中最后一个参数是一个fasync_struct结构的二维指针，一般设备驱动里
应该定义一个特定file相关的fasync_struct的指针，用于保存内核分配的fasync_struct
的地址, 其实前面所谓注册一个fasync_struct, 就是请求内核分配一个fasync_struct的
结构，然后返回该结构的地址给设备驱动。
```
int test_fasync(int fd, struct file *file, int mode)
{
	return fasync_helper(fd, file, mode, &async_queue);
}

```

当内核要发送信号给用户态进程的时候需要调用下kill_fasync函数, 该函数会向
fasync_struct对应的fd所绑定的进程发送一个SIGIO信号。这里的SIGIO也可以换成其他的
信号，但是一般用SIGIO信号发异步通知。
```
void test_trigger_sigio(struct timer_list *tm)
{
	kill_fasync(&async_queue, SIGIO, POLL_IN);
}

```
可以看到在测试程序里，我们为了方便测试，其实是起了一个内核定时器，在测试内核模块
加载10s后，向fd绑定的用户态进程发送一个SIGIO信号。(注意，这个测试程序是在主线
内核5.1上调试的，主线内核在4.15更新了内核定时器的API，这里用的是新内核定时器API)

在用户态程序中，需要设置设备fd，使其和当前进程绑定，还要使其可以接受fasync信号。
```
ret = fcntl(fd, F_SETOWN, getpid());
if (ret == -1) {
	printf("u fasync: fail to bind process\n");
	return -2;
}

fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | FASYNC);
if (ret == -1) {
	printf("u fasync: fail to set fasync\n");
	return -3;
}
```
这样设置后在内核调用kill_fasync，内核就知道把信号发给哪个进程。不同shell可能对
于信号设定有所不同，这里在测试之前先把SIGIO信号unmask，避免SIGIO被shell默认mask
掉，然后子进程继承父进程的信号设定，也mask住SIGIO的情况。

Linux信号的实现基本逻辑是在进程的signal pending表里标记其他进程或者内核给自己发
的信号，然后在进程从内核态切回用户态的时候再去扫描signal pending表以响应信号。
如果用户态进程一直没有系统调用，那么内核态发的SIGIO会不会得不到即使的响应? 另外
是否其他的内核活动也会引起测试进程发生内核态向用户态切换的过程，其中一个最可能
的情况就是内核周期性的调度。测试程序中做了一个简单的测试，用户态程序在设置好
fd后就进入死循环，而内核设备驱动在加载10s后会给用户态进程发一个SIGIO信号。测试
的结果是用户态进程可以在10s左右收到内核发的SIGIO信号。
