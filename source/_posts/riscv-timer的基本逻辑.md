---
title: riscv timer的基本逻辑
tags:
  - riscv
  - 计算机体系结构
  - QEMU
description: 本文梳理riscv上timer的基本软硬件逻辑，硬件模型基于qemu，使用的qemu版本是6.2.0， 内核代码分析使用的版本是v5.12-rc8。本文尽可能多的梳理riscv上timer相关的内容，需要独立描述的会在文中指出来。
abbrlink: 1040
date: 2022-08-11 21:08:08
categories:
---

基础逻辑
---------

 riscv上有两个最基本的中断控制器aclint和plic，前者的全称是Advanced Core Local
 Interruptor，是核内的中断控制器，主要是用来产生timer中断和software中断，后者的全称
 是Platform Level Interruptor Controller，主要用来收集外设的中断，plic通过外部中断
 向CPU报中断。

 timer相关的寄存器以及寄存器域段有: mip/mie里和timer相关的域段，mtime以及mtimecmp。
 mie里有控制timer中断使能的bit: MTIE/STIE，控制M mode和S mode timer interrupter是否
 使能，mip里有表示是否存在pending的timer中断的bit: MTIP/STIP。

 mtime是一个可读可写的计数器，其中的数值以一定的时间间隔递增，计数器计满后会回绕，
 mtimecmp寄存器里的数值用来和mtime做比较，当mtime的值大于等于mtimecmp的值，并且
 MTIE使能时，M mode timer中断被触发。

 软件可以在timer中断处理函数里，更新mtimecmp的值，从而维持一个固定周期的时钟中断，
 一般这个中断就是Linux内核的时钟中断。软件可以写STIP触发一个S mode timer中断。

 aclint上把mtime以及mtimecmp抽象成一个M mode timer这样的设备，mtime和mtimecmp是这个
 设备上的MMIO接口，一个M mode timer上有一个mtime，一个M mode timer为服务的每个hart
 设置一个mtimecmp。aclint协议上描述，一个系统可能会有多个M mode timer设备，这样做
 的目的是在CPU存在分层拓扑的时候，比如CPU cluster/node/socket时，一组CPU可以和一个
 M mode timer做在一起，方便这一组CPU的功耗管理。在系统中有多个M mode timer时，需要
 做多个mtime数值上的同步，使得多个mtime之间的误差在一定范围之内。

 alint只定义了M mode timer, riscv的sstc扩展定义了S mode下的timer。对于S mode timer，
 sstc只在每个hart上增加了stimecmp寄存器，当time计数值大于等于stimecmp时触发S mode
 timer中断，stimecmp是一个CSR寄存器。从timer整体定义上看，riscv这里定义的比较乱，
 M mode timer抽象成一个外设，接口是MMIO，但是S mode timer却改成了CSR，而且sstc还
 修改了riscv特权级spec里的一些系统寄存器的定义，mip.STIP这个域段在stimecmp有无时，
 读写属性是不一样的，当支持S mode但是没有实现stimecmp时，mip.STIP是读写的，写这个
 bit会触发S mode timer中断，当实现stimecmp时，mip.STIP是只读的。这样的实现意味着，
 在有S mode timer的系统上，将无法使用M mode timer从M mode通过mip.STIP触发S mode
 timer中断，相关的软件方案需要随之变动。sstc里还有虚拟化相关的描述，这些需要在独立
 文档中描述。

 riscv上还定义了一个用户态可以访问的计数器RDTIME，这个计数器从开机起就以一定的频率
 一直递增。

qemu逻辑 - M mode timer
------------------------

 qemu中的aclint和plic的代码路径分别在：hw/intc/riscv_aclint.c和hw/intc/sifive_plic.c。
 这里我们只关注和timer相关的部分，可以看到在riscv_aclint.c里只有M mode timer中断的
 触发代码。

 在qemu上跑内核的时候，发现总是一个M mode timer中断跟着一个S mode timer中断，然后
 再跟一个S mode ecall。qemu里并没有触发S mode timer的代码，可以猜测S mode timer中断
 是在opensbi里触发的。

 整个逻辑是：当mtime大于等于mtimecmp时触发一个M mode中断，opensbi里的中断处理逻辑
 会写STIP，由于S mode time中断已经被委托到S mode处理，在M mode返回S mode后，S mode
 timer中断就会被触发。
```
 /* opensbi/lib/sbi/sbi_trap.c */
 sbi_trap_handler
   +-> sbi_trap_noaia/ais_irq
     +-> sbi_timer_process
           /* 如果没有SSTC特性，才这样处理 */ 
       +-> csr_set(CSR_MIP, MIP_STIP)
```
 S mode timer中断处理函数里通过S mode ecall写mtimecmp，为下一次M mode timer中断配置
 合理的数值。这里面可能有一个问题，mtime如果触发中断后不往前走，就会有时间上的误差，
 可以想象，如果mtime在中断触发后依然往前走，就不会有这个问题。

 查看qemu aclint的代码，其中使用QEMU_CLOCK_VIRTUAL这个时钟来计算mtime寄存器里的值,
 而QEMU_CLOCK_VIRTUAL是来自host上获取时间的函数clock_gettime/get_clock_realtime，
 获取的是一个不断流逝的时间值。

 说到虚拟机的timer，就有一个问题要问，虚拟机里的时间和真实世界里的时间数值上一样么？
 我们看下aclint驱动配置timecmp时，qemu里的timer中断定时是怎么处理里。
```
 riscv_aclint_mtimer_write
   +-> riscv_aclint_mtimer_write_timecmp
     +-> ns_diff = muldiv64(diff, NANOSECONDS_PER_SECOND, timebase_freq)
```
 如上，使用传进来的timecmp值，计算出timer中断间隔的实际值，然后在host上启动一个
 定时器来做模拟，timebase_freq是mtime寄存器作为counter的频率，所以实际时间的计算
 就是：(mtimecmp - 上一个中断点的mtime值) * 1/timebase_freq * NANOSECONDS_PER_SECOND，
 单位是ns，就是上面muldiv64函数的计算结果。所以，虚拟机看到的时间和实际时间是一样的。

qemu逻辑 - S mode timer
------------------------

 qemu里S mode timer的代码在：target/riscv/time_helper.c，stimecmp是CSR寄存器，所以
 stimecmp的访问代码在: target/riscv/csr.c。

 可以看到stimecmp的读写逻辑和timecmp的读写逻辑基本上是一致的，而时钟源都使用一样的
 QEMU_CLOCK_VIRTUAL。具体的看，针对sstc，qemu在time_helper.c里实现了hw/intc/riscv_alint.c
 里很多一样的逻辑，通过CPU env里的rdtime_fn_arg/rdtime_fn拿到riscv_alint.c里定义
 的和时钟源相关的信息，比如，timebase_freq。

qemu逻辑 - RDTIME
------------------

 这个寄存器在U mode也可以访问，所以具体实现也是qemu user mode和system mode各自支持的。

 qemu user mode下的实现，直接使用host上的特性tick执行实现，这样的实现感觉真心
 没有什么实用价值。

 system mode下的实现，RDTIME的时钟源也是QEMU_CLOCK_VIRTUAL，读相关的CSR直接得到
 计算后的计数值，数值上和如上M/S mode timer计数器的值是一样，表示自系统启动以来
 的一个计数值。所以，这个值和墙上时钟还是不一样的，墙上时钟在系统关机后会使用主板
 上的纽扣电池继续运行。

Linux内核逻辑
--------------

 内核timer初始化在：arch/riscv/kernel/time.c的time_init。内核相关驱动的位置在：
 drivers/clocksource/timer-riscv.c。截止到目前为止，S mode timer的内核patch还在
 社区评审，相关的连接在lwn.net/Articles/886863/。

 riscv timer的内核驱动把对应的of_device_id静态定义到__timer_of_table段里。time_init
 里的timer_probe会扫描__timer_of_table这个段里timer相关的of_device_id, 然后调用
 对应的初始化函数，这里就是调用riscv_timer_init_dt，这个函数会找见intc对应的domain，
 通过domain和S mode timer硬件中断号得到S mode timer对应中断的virq，最后向virq注册
 S timer的中断处理函数：
```
 riscv_timer_init_dt
   +-> riscv_clock_event_irq = irq_create_mapping(domain, RV_IRQ_TIMER)
   +-> request_percpu_irq(riscv_clock_event_irq, riscv_timer_interrupt, ...)
```

 riscv timer驱动会注册clocksource和clock_event_device。

 clocksource就是指不断递增的时钟源，比如riscv上的mtime寄存器以一定的频率增加计数，
 它就是一个clocksource. riscv_clocksource提供一个read接口，通过这个接口可以得到
 当前mtime计数器里的值。在riscv qemu virt平台上这个计数器的频率定义在cpus节点的
 timebase-frequency字段，它的值是0x989680，就是十进制的10000000，也就是说这个计数
 器10ns计数一下。

 clock_event_device指的是可以产生和时钟相关事件的device，比如riscv上当mtime的值
 大于等于mtimecmp的值时会上报一个M mode timer的中断，mtime和mtimecmp就可以被看作
 一个clock_event_device。riscv timer驱动里定义的struct clock_event_device riscv_clock_event
 是个per CPU变量，timer的中断处理函数就是调用riscv_clock_event里的event_handler。
 但是这个回调函数并不是在驱动里直接提供的。

 event_handler的注册逻辑是通过这个驱动里注册的cpu hotplu回调函数riscv_timer_starting_cpu
 完成的，大概的调用逻辑如下，可以看见event_handler实际上是一个公共函数tick_handle_periodic。
```
start_kernel
  +-> time_init
    +-> timer_probe
      +-> cpuhp_setup_state
        +-> __cpuhp_setup_state
          +-> __cpuhp_setup_state_cpuslocked
            +-> cpuhp_issue_call
              +-> cpuhp_invoke_callback
                +-> riscv_timer_starting_cpu
                  +-> clockevents_config_and_register
                    +-> clockevents_register_device
                      +-> tick_check_new_device
                        +-> tick_setup_device
                          +-> tick_setup_periodic
			        /* event_handler回调 */
			    +-> tick_handle_periodic
```
 tick_handle_periodic就是每次时钟中断时要运行的逻辑，相关逻辑已经和调度有关系，
 在另外讲调度的文章里独立分析吧。

 实际上，tick_handle_periodic只是在内核开始的时候用，内核随后会把event_handler切
 到高精度定时器的回调函数上hrtimer_interrupt，这个函数里只是处理时间相关的东西，
 tick时发生的调度要回到entry.S里，也就是处理完timer中断再处理调度相关的逻辑。
