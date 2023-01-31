---
title: Linux中断学习笔记1
tags:
  - Linux内核
description: N年前学习Linux中断的一个笔记，当时要支持GPIO做为中断使用
abbrlink: aaceb51e
date: 2021-07-17 11:12:26
categories:
---

硬件
----

  cpu在正常执行一条条指令的时候，可以由于某些原因跳到一些异常处理程序，执行一些
  代码，然后再返回原来的程序运行。按照触发异常的源头，异常包括：

1. 中断，异步的由外部器件发起，cpu响应后，保存现场，然后跳到特定位置执行，然后返回
   中断处继续执行下面的程序

2. 同步中断，由系统中一些错误的指令引起（比如除零，非法指令等），是和cpu同步的。
   软中断（实现系统调用的地方）就是人为的通过指令来产生一个异常，然后cpu切换工作模式，
   对应的进入内核空间开始执行代码

  中断系统的整个硬件包括：cpu,中断控制器，外设。在arm体系中这里的cpu指的是cpu核，
  中断控制器一般是标准的gic,通常gic已经被Soc厂商集成在Soc中了, gic的输出接cpu核
  上的irq和frq引脚, gic的输入接Soc内各ip模块的中断产生引脚, ip模块对应的中断线
  （gic上的输入引脚）在Soc内已经定死。不清楚gic的输入引脚可否直接连Soc的io管脚
```
  Soc pin |  in Soc:
   \-->      -|--ip模块中断控制---->gic中断控制---->cpu中断控制  
              | 
```
  一个中断从发起到cpu接收到的流程可以用上面的图来表示,需要配置各个中断控制中的
  相关寄存器，使得中断在物理上被cpu接收到

中断实现 
--------

  linux内核和中断相关的核心数据结构有：struct irq_desc (linux/irqdesc.h), 每个
  中断号对应着一个这样的结构，所有的irq_desc以数组或是树的形式存在
```
  struct irq_desc
      |--> struct irq_data irq_data -----> |-> mask,irq,hwirq,state_use_accessors
      |                                                           |-> struct irq_chip *chip
      |                                                           |-> struct irq_domain *domain 
      |                                                           |-> void *handler_data, *chip_data...
      |--> irq_flow_handler_t handle_irq   
      |--> struct irqaction *action ---->|-> irq_handler_t handler, thread_fn
      |--> raw_spinlock_t lock           |-> void *dev_id, int irq, flags
      |--> struct proc_dir_entry *dir    |-> unsigned long thread_flags, thread_mask
      |--> const char *name...           |-> const char *name
      |-> struct proc_dir_entry *dir...
```
  对着上面的数据结构分析中断流程:(arm体系)

  中断被cpu接收到之后，汇编程序处理后调用的第一个程序是：
  asm_do_IRQ(unsigned int irq, struct pt_regs *regs)
```
        -->handle_IRQ(irq, regs);
             -->struct pt_regs *old_regs = set_irq_regs(regs);
             irq_enter();
             generic_handle_irq(irq);
                  -->generic_handle_irq_desc(irq, desc);
                             -->desc->handle_irq(irq, desc);
     irq_exit();
     set_irq_regs(old_regs);
```

  最后调用的 handle_irq() 是注册在对应irq_desc中的的中断流函数, 处理中断嵌套等问题
  一般电平中断用标准的: handle_level_irq() (kernel/irq/chip.c)
      边沿中断用标准的: handle_edge_irq(), 上面的chip.c还有另外几种中断流函数,接着
  中断流函数向下分析:
```
  handle_edge_irq(unsigned int irq, struct irq_desc *desc)
      -->raw_spin_lock(&desc->lock);
      ...
     desc->irq_data.chip->irq_ack(&desc->irq_data); 
     ...
    handle_irq_event(desc);
     -->struct irqaction *action = desc->action;
     raw_spin_unlock(&desc->lock);
     handle_irq_event_percpu(desc, action);
     -->...
            action->handler(irq, action->dev_id);
          ...
     raw_spin_lock(&desc->lock);
 raw_spin_unlock(&desc->lock);
```
  
  最后调用的 handle()就是request_irq中注册的中断处理函数, 对应irq_desc中的action 
  中的handler。上面的中断流函数中调用了irq_ack(), irq_ack()是注册在irq_chip中的
  函数, 最上面的数据结构显示irq_chip在irq_desc中的irq_data中
```
  struct irq_chip {
	    ...
	void (*irq_enable)(struct irq_data *data);
	void (*irq_ack)(struct irq_data *data);
	void (*irq_mask)(struct irq_data *data);
	void (*irq_unmask)(struct irq_data *data);
	void (*irq_eoi)(struct irq_data *data);
	int (*irq_set_type)(struct irq_data *data, unsigned int flow_type);
	    ...
	unsigned long flags;
  }
```
  irq_chip包括和硬件相关的一组回调函数，他们直接操作中断相关的寄存器, 比如irq_enble
  使能中断线，irq_mask屏蔽中断线

中断使用
--------

  在驱动程序中只需要调用request_irq()注册中断处理程序即可使用中断，上面的中断实现
  为我们做了很多工作。注册中断和释放中断：
```
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
const char *name, void *dev)
    request_irq有可能会睡眠，里面会调用kmalloc()分配内存
    free_irq(unsigned int irq, void *dev_id)
```

  中断处理函数: irqreturn_t handler(int irq, void *dev_id) 无需要可重入，一个中
  断在执行的时候，相同的中断在其他的所有的处理器上的中断线都会被屏蔽?

  request_irq()中使用中断号irq注册中断处理函数, 一个中断号也可以注册多个处理函数。
  中断发生了，cpu就跳去执行中断处理函数，一个外设也可以发出几个不同的中断, 这时
  可以在外设的驱动程序中注册多个中断处理函数

  当外设的中断引脚和gic的输入直接相连时,直接注册中断处理函数就可以使用中断。这是
  因为gic的驱动程序(drivers/irqchip/irq-gic.c)已经实现了开头的数据结构和回调函数,
  其中包括：gic对应的irq_chip中的回调函数(这些回调函数操作gic的输入，可以mask,
  unmask, enable等等?), gic对应的中断流函数(gic_handle_irq()?)

  若是Soc内部的ip存在中断控制，比如gpio, 它一端接N个输入管脚，每个输入管脚可以接
  收中断，它的中断输出接一个gic的输入，也就是说gpio的N个管脚上的中断输入，都反应
  在gic的一个输入上。gpio中也有一组寄存器可以控制中断(enable, mask, unmask gpio
  输入管脚上的中断)。这时我们要自己为这些中断建立开头的那写数据结构和回调函数

中断流函数
----------

  以一个gpio控制器的驱动程序为例说明，linux内核中各个厂商的gpio驱动在/drivers/gpio
  内, 以gpio-mvebu.c来分析

  irq = platform_get_irq(pdev, i) 取出gpio对应的中断号，这个中断号是系统一开是就
  定好的，是gic给gpio分配的中断号，这一个中断号可以对应gpio的多个输入。当gpio
  引脚配置成中断引脚时，任何一个引脚上产生的中断都通过这个中断号上报给gic
```
irq_set_handler_data(irq, mvchip)
   -->desc->irq_data.handler_data = data;

irq_set_chained_handler(irq, mvebu_gpio_irq_handler)
   -->__irq_set_handler(irq, handle, 1, NULL); // 1 表示：is_chained
   desc->handled_irq = handle;
         
mvchip->irqbase = irq_alloc_descs(-1, 0, ngpios, -1)
__irq_alloc_descs(irq, from, cnt, node, THIS_MODULE)
   -->start=bitmap_find_next_zero_area(allocated_irqs,IRQ_BITMAP_BITS,from,cnt,0);
            bitmap_set(allocated_irqs, start, cnt);
   alloc_descs(start, cnt, node, owner);
    以上分配了ngpios个连续的中断号，最后返回的是第一个中断号
```
```
gc = irq_alloc_generic_chip("mvebu_gpio_irq", 2, mvchip->irqbase,
   mvchip->membase, handle_level_irq); 
   -->struct irq_chip_generic *gc;
   unsigned long sz = sizeof(*gc) + num_ct * sizeof(struct irq_chip_type);
   gc = kzalloc(sz, GFP_KERNEL);
            irq_init_generic_chip(gc, name, num_ct, irq_base, reg_base, handler);
        --> raw_spin_lock_init(&gc->lock);
                gc->num_ct = num_ct;
                gc->irq_base = irq_base;

                gc->reg_base = reg_base;

                gc->chip_types->chip.name = name;

                gc->chip_types->handler = handler;
```
  以上动态分配了一个irq_chip_generic结构, 然后用之前分配的连续中断号中的第一个
  中断号填充irq_base, 并且填充中断流函数
```
    stuct irq_chip_generic
    |--> irq_base
    |--> u32 mask_cache, tpye_cache
    |--> void *private
    |...
    |--> struct irq_chip_type chip_types[0]--->|--> struct irq_chip
    |                                                                         |--> irq_flow_handler_t handler
                                                                              |--> type
                                                                              |...
gc->private = mvchip; // gc: *irq_chip_generic 
ct = &gc->chip_types[0]; // ct: *irq_chip_type
ct->type = IRQ_TYPE_LEVEL_HIGH | IRQ_TYPE_LEVEL_LOW;
ct->chip.irq_mask = mvebu_gpio_level_irq_mask;
ct->chip.irq_unmask = mvebu_gpio_level_irq_unmask;
ct->chip.irq_set_type = mvebu_gpio_irq_set_type;
ct->chip.name = mvchip->chip.label;
```
  以上向irq_chip中注册了相应的回调函数，这些回调函数处理gpio中断相关，如mask,
  ack, unmask gpio中断线。现在irq_desc结构也分配好了, irq_chip中的回调函数也注册
  好了, 需要做的是向irq_desc中注册相应的irq_chip
```
irq_setup_generic_chip(gc, IRQ_MSK(ngpios), 0,
      IRQ_NOREQUEST, IRQ_LEVEL | IRQ_NOPROBE);
        -->list_add_tail(&gc->list, &gc_list);
irq_set_chip_and_handler(i, chip, ct->handler); // for loop to do this
irq_set_chip_data(i, gc);
```
  以上先把上面动态生成的irq_chip_generic结构加入gc_list链表，然后向之前申请的
  irq_desc注册各自的irq_chip和中断流函数

  mvchip->domain = irq_domain_add_simple(); 
  添加一个irq_domain, 作用暂时不清楚
