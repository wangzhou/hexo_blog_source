---
title: riscv kvm中断虚拟化的基本逻辑
tags:
  - riscv
  - kvm
  - 虚拟化
  - 中断
description: 本文分析riscv kvm中断虚拟化的相关逻辑，分析基于qemu v7.0.0，Linux内核基于v5.19。 中断直通没有分析。
abbrlink: 24563
date: 2022-09-07 18:52:57
categories:
---

基本逻辑
---------

 kvm虚拟化的时候，guest的代码直接运行在host上，怎么样触发虚拟机中断是一个问题。
 在非虚拟化的时候，中断的触发是一个物理过程，中断被触发后，跳到异常向量，异常向量
 先保存被中断的上下文，然后执行异常向量的业务代码。但是，kvm虚拟化的场景，所谓虚拟机
 只是运行的线程，我们假设硬件可以直接触发中断，但是触发中断的时候，物理CPU都可能
 运行的是其他的虚拟机，怎么把特定虚拟机上的中断投递正确，这是一个需要解决的基本问题。

 我们再看另一个场景，在kvm虚拟化的时候，系统里有一个完全用软件模拟的IO设备，比如，
 一个网卡，那这个网卡的中断怎么传递给正在运行的虚拟机。从上帝视角看，运行kvm虚拟机
 就是在kvm ioctl(KVM_RUN)里切到虚拟机的代码去执行，要打断它，一个自然的想法就是从
 qemu里再通过一定的方法“注入”一个中断，可以想象，所谓的“注入”中断，就是写一个虚拟机
 对应的寄存器，触发这个虚拟机上的执行流跳转到异常向量。

 对于核内的中断，比如时钟中断，还可以在host kvm里就直接做注入。比如，可以在kvm里
 起一个定时器，当定时到了的时候给虚拟机注入一个时钟中断。

 下面具体看下当前riscv的具体实现是怎么样的。

时钟中断
---------

 在kvm vcpu创建的时候，为vcpu创建timer，实现上就是创建一个高精度定时器，定时到了
 的时候给vcpu注入一个时钟中断。
```
 /* linux/arch/riscv/kvm/vcpu.c */
 kvm_arch_vcpu_create
   +-> kvm_riscv_vcpu_timer_init
         /* 中断注入的接口被封装到这个函数里，中断注入接口是kvm_riscv_vcpu_set_interrupt */
     +-> t->hrt.function = kvm_riscv_vcpu_hrtimer_expired
```
 可以看到注入中断实际上是给kvm管理的vcpu软件结构体的irqs_pending/irqs_pending_mask
 bit置一，后面这个vcpu实际换到物理cpu上执行的时候，再写相应的寄存器触发cpu中断。

 kvm同时通过ioctl接口向qemu提供一组timer寄存器的读写接口，看起来，qemu会在vm stop
 的时候把timer的这组寄存器读上来，在vm resume的时候把这组寄存器写下去，这样在vm
 停止的时候，vm的timer状态就是没有变化的。
```
 /* linux/virt/kvm/kvm_main.c */
 kvm_vcpu_ioctl
       /* linux/arch/riscv/kvm/vcpu.c, riscv里这个ioctl用来配置和读取寄存器，其中就包括timer相关寄存器的操作 */
   +-> kvm_arch_vcpu_ioctl
     [...]
           /*
            * 这里只看下timer寄存器的配置接口, 如下的函数里就涵盖了
            * frequency/time/compare/timer启动的配置，其中timer启动的实现就是启动
            * 上面提到的高精度定时器
            */
       +-> kvm_riscv_vcpu_set_reg_timer
```
 
 qemu里在target/riscv/kvm.c里把timer寄存器配置的函数注册到了qemu里：
```
 kvm_arch_init_vcpu
   +-> qemu_add_vm_change_state_handler(kvm_riscv_state_change, cs)
```
 在vm_state_notify里调用，看起来vm_state_notify是在vm启动停止的时候才会使用，用来
 做kvm和qemu里的信息的同步。

 我们再考虑一个问题，怎么做到虚拟机的时间和实际时间相等。可以想象，只要让模拟vcpu
 timer的那个定时器一直跑就可以，每次都把时间更新到vcpu的数据结构里就好。在每次停止
 和启动vm的时候，把时间在kvm和qemu之间做同步，vm停止的时候，vm的timer就也是停下来
 的。在qemu中date下获取当前时间，然后在qemu monitor里停止vm，过一会后启动vm，可以
 看见，时间基本是没有改变的。

外设中断注入
-------------

 完全虚拟的设备向kvm注入中断时，虚拟设备这一层发起中断的流程还和tcg下的一样，到了
 向vcpu触发中断这一步，没有向tcg那样写到vcpu在qemu的结构体里，因为写到这个结构体
 里丝毫不会对kvm运行的指令有影响，这一步使用kvm_riscv_set_irq向kvm里注入中断，
 这个函数的封装函数在target/riscv/cpu.c里注册：
```
 /* qemu/target/riscv/cpu.c */
 riscv_cpu_init
       /* riscv_cpu_set_irq里会调用kvm_riscv_set_irq */
   +-> qdev_init_gpio_in(DEVICE(cpu), riscv_cpu_set_irq, ...)
```

 kvm_riscv_set_irq使用ioctl VM_INTERRUPT注入中断，基本的kvm里的调用流程如下:
```
 /* linux/virt/kvm/kvm_main.c */
 kvm_vcpu_ioctl
       /* linux/arch/riscv/kvm/vcpu.c */
   +-> kvm_arch_vcpu_async_ioctl
     +-> kvm_riscv_vcpu_set_interrupt
```

中断直通
---------

 riscv现在还没有中断直通，ARM的GICv4支持了这个功能，太复杂了，再独立的文章中分析吧。
