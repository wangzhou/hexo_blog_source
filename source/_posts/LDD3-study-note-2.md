---
title: LDD3 study note 2
tags:
  - 读书笔记
  - LDD3
description: >-
  这篇文章在note1的基础上，进一步记录实现一个ioctl要注意的地方。相关的代码在：
  https://github.com/wangzhou/scull.git, tag: scull_2
categories: read
abbrlink: 715ce853
date: 2021-07-05 22:39:58
---

ioctl
--------

驱动可以通过ioctl函数定义一组和用户态程序交互的接口.
ioctl的用户态接口是：int ioctl(int d, int request, ...). 一般我们可以认为是：
int ioctl(int fd, int cmd, int arg)。其中fd是要操作的文件，cmd是下发的命令字,
和驱动里ioctl实现的命令字是一一对应的。arg是传入内核的参数，可以看到一般的情况
下arg这个参数是一个指针变量。

命令码
---------

cmd不是可以随便定义的，具体可以参考linux/Documentation/ioctl/ioctl-number.txt
这个文档。简单来讲，一个命令字是四段组成的。每段具体是什么内容可以查看LDD3或者
上面的文档。生成命令字需要用内核提供的一组宏，这组宏的定义在：
linux/include/uapi/asm-generic/ioctl.h

一般定义命令字用下面这组宏:
```
#define _IO(type,nr)		_IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)	_IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOW(type,nr,size)	_IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOWR(type,nr,size)	_IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
```

type是在ioctl-number.txt里讲过的魔术字，作为命令字最基本的区分。nr是ioctl接口
的第几个命令，一般是0,1,2..., size LDD3上解释的不是很清楚，暂时用int(Fix me)。
命令的类型会合成命令字中的一段。这样，以上四段内容会拼成一个ioctl的命令字。

可以看到一个ioctl支持的接口命令中type，size一样的，命令的类型和序号有可能有区别。

用户态头文件
---------------

ioctl是一种用户态和内核交流信息的方式。用户态调用ioctl的时候，发起的命令、传入
内核的数据结构, 在用户态都要有定义。目前的本人的做法是，在用户态的头文件中拷贝
内核中命令码生成的相关宏定义(Fix me)，对于交流信息的数据结构，也在用户态头文件中
再定义一次。

传变量和传指针
-----------------

受到ioctl接口的限制, 用户态可以使用传变量和传指针的方式向内核发送信息, 传变量
也只能是一个unsigned long类型。内核向用户态传信息，就只能向用户态传进来的指针里
写数据了。
