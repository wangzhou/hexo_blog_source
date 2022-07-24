---
title: 用户态下使用gpio中断
tags:
  - Linux内核
  - gpio
description: >-
  linux内核中的gpio驱动可以使用内核中提供的gpio驱动框架来实现(drivers/gpio/gpiolib.c)
  该框架使用sys文件系统把gpio暴露给用户态程序使用，本文介绍怎么在用户态下使用 gpio提供的中断功能。在介绍使用的同时，介绍一些涉及到的内部的实现过程
abbrlink: 99e8ef41
date: 2021-07-17 11:09:20
categories:
---

使用
----

确定你的gpio驱动是好用的，这时会在/sys/class/gpio下发现gpio对应的文件:gpio***,
export, unexport, gpio***是gpio控制器对应的文件，export, unexport是gpio框架
提供的用来向用户态导出gpio的文件

假设使用66号gpio口作为中断端口，即产生中断的器件的中断管脚连接的是soc的66号gpio
管脚。echo "66" > export 向外导出管脚，会发现/sys/class/gpio下多了目录gpio66
gpio66目录中有文件value, direction, edge, power, device等等

echo "in" > direction 设置gpio66脚为输入
echo "falling" > edge 设置gpio66脚为下降沿触发中断, 也可以把falling改成rising
即为上升沿触发，这时当gpio66管脚上存在一个falling时就会接收到一个中断，怎么把
这个接收到的中断在用户态反应出来呢？

在接收到中断的时候value的值会从原来的1变成0，这里假设是下降沿触发, 所以可以
使用poll()函数阻塞在value文件对应的文件描述符上，当文件发生变化的时候poll返回
相应的中断，具体代码:
http://blog.skerpa.com/dschnell/blog/2013/10/27/linux-and-gpio-userspace-interrupts/
...

gpiolib.c中的实现
-----------------

对edge的读写，最后会调用到： gpio_edge_show()，gpio_edge_store(), 可以看看
echo "falling" > edge 内核的函数调用

内核首先找到falling对应的编码，然后在gpio_setup_irq中注册中断处理函数:
gpio_sysfs_irq, 所以当gpio管脚上发生中断时，最后会调用中断处理函数中的
wake_up_interrupt()通知对应文件上的等待队列?

request_irq(), free_irq()函数在注册中断，释放中断的时候，会对相应的中断线
做一定的处理, 包括使能中断等

但是有一个东西不清楚：假设是一个下降沿触发的中断，在接收到中断的时候对应的
/sys/class/gpio/gpio***/value 中的值应该从1变成0, 但是使用上面博客中的代码
可以发现value中的值一直是1，手动cat value发现其中的值是0。依然没有找见value
被设置为0的对应代码?
