---
title: qemu tcg中断模拟
abbrlink: 8198
date: 2022-07-25 15:44:42
tags: [QEMU, 中断]
description: "记录qemu irq相关的代码逻辑，以riscv为例。分析使用的qemu代码的版本是7.1.50。"
categories:
---

qemu的整个中断处理流程可以大概分为核内和核外，核内指的是CPU核在收到中断信号时的
处理逻辑，核外我们可以认为就是指各种中断控制器的处理逻辑。

qemu tcg整体的翻译执行流程大概如下，这里描述的是核内模拟的顶层逻辑。
```
 setjmp;

 while (检查并处理中断或异常) {
        前端翻译;
        后端翻译;
        执行host执行改变guest CPU状态; // 有异常时longjmp到setjmp处
 }
```
所以，实际响应并处理中断总是在一个tb翻译执行完。qemu有chained tb，如果一直在chained
tb里跳来跳去，那不是一直响应不了中断，为此qemu里的中断可以触发跳过一个tb的执行，
这样，一旦有中断来了，就可以快速响应，下面会介绍这个地方。

riscv上核内和核外的中断信号接口用gpio管脚来模拟。CPU核通过如下函数注册一个向cpu
写入中断的接口。向一个gpio上写入信息，逻辑上应该触发gpio handler被调用一次。
```
/* target/riscv/cpu.c */
qdev_init_gpio_in(DEVICE(cpu), riscv_cpu_set_irq, 12)
```

从逻辑上分析，应该是中断控制器调用这个接口把中断写入到cpu。riscv上有多个中断控制
器: plic, aclint以及AIA, qemu上它们的模拟都在/hw/intc/下，对应文件分别是：sifive_plic.c，
riscv_aclint.c，riscv_aplic.c和riscv_imsic.c，其实AIA被实现为最后两个文件。plic/aplic
和外部设备相关，aclint是核内部的中断控制器，主要产生时钟中断和SWI，imsic则是为了
支持MSI和虚拟化中断直通。

中断控制器里使用qdev_init_gpio_out创建一个输出的GPIO，然后使用qdev_connect_gpio_out
把输出GPIO和输入的GPIO连在一起，在中断控制器里使用qemu_set_irq/qemu_irq_raise向
CPU发中断，本质上是触发riscv_cpu_set_irq被调用到。至于gpio_in对应的qemu_irq.handler
如何被gpio_out看见，就是qemu_set_irq/qemu_irq_raise如何触发riscv_cpu_set_irq被调用，
其中的细节分析，我们放到本文的最后。

如上，我们已经把核内和核外中断的大概逻辑梳理了下，下面重点看CPU核接收到中断的逻辑。
riscv上中断控制器的qemu模拟需要在独立的文档中描述。

riscv_cpu_set_irq修改riscv系统寄存器mip，把中断类型写入到里面。
```
riscv_cpu_set_irq
     /* 可以看到这里是把中断无条件写入到mip里的 */
  -> riscv_cpu_update_mip
       /*
        * 写cpu->interrupt_request, 大循环里的cpu_handle_irq根据这个判断是否
        * 要处理中断
	*/
    -> cpu_interrupt
      -> cpu->interrupt_request |= mask;
      -> qatomic_set(&cpu_neg(cpu)->icount_decr.u16.high, -1);
```
注意写icount_decr.u16.high的这个逻辑。qemu在一个tb的前端翻译的前后会插入一段翻译
的代码，相关逻辑在gen_intermediate_code->translator_loop->gen_tb_start/gen_tb_end，
gen_tb_start中产生的逻辑会检测如上icount_decr.u16.high的值，如果比0小，就会直接跳到
tb最后的一个label处，这个label在gen_tb_end里配置，这样，中断一旦写了icount_decr.u16.high
这个标记，tb执行的时候就会跳过当前tb的业务代码，相当于空执行了一个tb。这个跳转直接
跳过了chained tb的地方，所以可以从chained tb里出来。

如上是中断上报CPU的模拟逻辑，报到CPU的中断需要在qemu翻译执行的大逻辑里处理，下面
描述这部分内容。上面一开始提到的“检查并处理中断或异常”处理中断的具体函数是cpu_handle_irq。
```
/*
 * cpu_handle_irq里调用cpu_exec_interrupt回调函数处理中断，riscv_cpu_exec_interrupt
 * 从mip里拿到中断号, 最后调用riscv_cpu_do_interrupt处理中断。riscv_cpu_exec_interrupt在
 * target/riscv/cpu_helper.c
 */
riscv_cpu_exec_interrupt
     /* 拿到中断号, 具体的逻辑我们可以在下面打开看下 */
  -> interruptno = riscv_cpu_local_irq_pending(env);
     /*
      * 注意这里会配置exception_index这个值，这个值在异常处理
      * cpu_handle_exception里也会读
      */
  -> cs->exception_index = RISCV_EXCP_INT_FLAG | interruptno;
     /* riscv在这个里一并处理异常和中断，改变机器的状态和pc，最后清exception_index */
  -> riscv_cpu_do_interrupt(cs);
       /* RISCV_EXCP_NONE是-1 */
    -> cs->exception_index = RISCV_EXCP_NONE
```

可以看见如下的逻辑先计算出各个特权级中断的enable状态，然后用enable状态和cpu中已
保存的pending状态得到实际要处理的中断(A处的pending表示)，注意这个是一个临时变量
CPU中保存的pending依然在。随后，根据中断优先级和中断委托逻辑得到要真实触发的中断。
```
static int riscv_cpu_local_irq_pending(CPURISCVState *env)
{
    int virq;
    uint64_t irqs, pending, mie, hsie, vsie;

    /* Determine interrupt enable state of all privilege modes */
    if (riscv_cpu_virt_enabled(env)) {
	/* n: cpu目前处于v状体，所以，高特权级m/hs的中断是打开的 */
        mie = 1;
        hsie = 1;
	/*
	 * n: 对于vs的中断，当cpu现在状态是vs，就看SIE的配置，当cpu现在是vu，
	 *    高特权级中断一定是开着，所以vs的中断是开着。
	 */
        vsie = (env->priv < PRV_S) ||
               (env->priv == PRV_S && get_field(env->mstatus, MSTATUS_SIE));
    } else {
	/*
	 * n: cpu不在v状体，所以，cpu可能在m，或者比m低，同样的逻辑，在m，就看
	 *    MIE的配置，比m低，那m中断一定是开着。
	 */
        mie = (env->priv < PRV_M) ||
              (env->priv == PRV_M && get_field(env->mstatus, MSTATUS_MIE));
	/*
	 * n: cpu不在v，cpu比S低，中断一定是开，cpu在S，就看SIE，cpu在M，
	 *    hs的中断应该是关的啊，为啥这里没有涉及？
	 */
        hsie = (env->priv < PRV_S) ||
               (env->priv == PRV_S && get_field(env->mstatus, MSTATUS_SIE));
	/* n: 因为cpu不在v，那cpu的特权级一定比vs高，vs中断一定是关的 */
        vsie = 0;
    }

    /* Determine all pending interrupts */
    pending = riscv_cpu_all_pending(env);     <------- A

    /*
     * n: 中断来了，这里就有优先级的处理，大的分类上，M > HS > VS, 就是说有了M,
     *    我们就先处理M的中断，依此类推。这里有一个问题，如果pending了多个中断，
     *    高特权级的中断先处理了，低特权级的中断pending还在么？
     *
     *    从实现上，中断pending记录在env->mip里，暂时没有看见哪里会清mip。
     */
    /* n: 中断有pending, 并且没有被代理出去，并且中断enable是开的，我们才计算中断号 */
    /* Check M-mode interrupts */
    irqs = pending & ~env->mideleg & -mie;
    if (irqs) {
        /* n: 计算一个特权级下最高优先级的中断 */
        return riscv_cpu_pending_to_irq(env, IRQ_M_EXT, IPRIO_DEFAULT_M,
                                        irqs, env->miprio);
    }

    /* Check HS-mode interrupts */
    irqs = pending & env->mideleg & ~env->hideleg & -hsie;
    if (irqs) {
        return riscv_cpu_pending_to_irq(env, IRQ_S_EXT, IPRIO_DEFAULT_S,
                                        irqs, env->siprio);
    }

    /* Check VS-mode interrupts */
    irqs = pending & env->mideleg & env->hideleg & -vsie;
    if (irqs) {
        virq = riscv_cpu_pending_to_irq(env, IRQ_S_EXT, IPRIO_DEFAULT_S,
                                        irqs >> 1, env->hviprio);
        return (virq <= 0) ? virq : virq + 1;
    }

    /* Indicate no pending interrupt */
    return RISCV_EXCP_NONE;
}
```

下面分析下，qemu_set_irq/qemu_irq_raise如何触发riscv_cpu_set_irq被调用。

qdev_init_gpio_in(DeviceState *dev, qemu_irq_handler handler, int n)给CPU核创建
n个中断gpio接口，每个中断gpio接口的中断处理函数都是handler，这里的dev就是CPU核的
父类，下面展开这个函数看下。
```
qdev_init_gpio_in
     /*
      * dev里有gpios，它是一个元素为NameGPIOList的链表，每个NameGPIOList是一组gpio，
      * NameGPIOList里直接嵌入了qemu_irq，每一个gpio都有一个，这样每个gpio都可以
      * 当一个中断管脚使用。
      *
      * 这个函数就是创建dev里n个这样的gpio, 这个n个gpio被一个NameGPIOList所管理。
      */
  +->qdev_init_gpio_in_named_with_opaque
```

qdev_init_gpio_out(DeviceState *dev, qemu_irq *pins, int n)给dev创建n个中断输出
接口，这里可以把dev理解为中断控制器。需要注意的是，qemu_irq本来已经是指针了，这里
把指针的地址送进来是为了后续配置qemu_irq这个指针的值。
```
qdev_init_gpio_out
  +-> qdev_init_gpio_out_named
        /*
	 * 给dev增加link属性，每个qemu_irq都增加了一个，link属性的名字是：unnamed-gpio-out[num],
	 */
    +-> object_property_add_link
```

qdev_connect_gpio_out_named(DeviceState *dev, const char *name, int n, qemu_irq input_pin)
这个函数在中断控制器驱动里被调用，dev是中断控制器，input_pin是CPU核上的中断gpio,
通过name得到上面中断控制器增加的link属性的名字。
```
qdev_connect_gpio_out_named
      /*
       * 这个函数把dev里的名字是propname的link和input_pin建立一个link。这里的关键
       * 就是要搞懂set_link的本质。这个函数内部会调用object一层的set函数，这个函数
       * 把link属性所对应的变量的地址直接设置成被指向目标的地址，具体就是把上面
       * gpio_out的qemu_irq换成了gpio_in的qemu_irq。这样，使用中断控制器的qemu_irq
       * 就可以直接触发CPU核的中断回调函数，因为经过这么link过后，中断控制器的qemu_irq
       * 已经就是CPU核的qemu_irq。
       */
  +-> object_property_set_link(OBJECT(dev), propname, OBJECT(input_pin), ...)
```

中断控制器里在处理外设上报来的中断后，一般用qemu_set_irq/qemu_irq_raise把中断继续
上CPU核上报。需要注意的是，这里的qemu_irq是中断控制器上的，而上面的逻辑中，中断控制
器上的qemu_irq只是和CPU核的qemu_irq建立了link，但是，这里入参qemu_irq就是CPU核一侧
qemu_irq的指针，这样qemu_irq_raise中直接调用qemu_irq的handler发起中断的逻辑就都通了。
```
qemu_irq_raise(qemu_irq irq)
  +-> qemu_set_irq
    +-> irq->handler
```
