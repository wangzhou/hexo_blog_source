---
title: qemu tcg中间码优化和后端翻译
tags:
  - QEMU
description: >-
  本文介绍qemu tcg下中间码翻译和后端翻译的基本逻辑，代码分析基于qemu v7.1.50，
  文中涉及的测试场景，我们选择riscv作为guest、arm64作为host。
abbrlink: 12053
date: 2023-03-15 22:22:00
categories:
---

基本逻辑
---------

 qemu tcg的基本翻译思路是把guest指令先翻译成中间码(IR)，然后再把IR翻译成host指令。
 guest->IR->host这种三段式实现的好处是把前端翻译，优化和后端翻译拆开了，降低了开发
 的难度，比如，要模拟一个新构架的CPU，只要实现guest->IR这一步就好，后续在X86或者在
 ARM64的host的机器上跑，重新编译下qemu就好，并不用去熟悉host CPU的构架。

 guest翻译成IR的逻辑在[这里]([https://wangzhou.github.io/qemu-tcg翻译执行核心逻辑分析/)已经有介绍，这一步主要靠程序员手写代码生成IR，本文主
 要讲中间码的优化和后端翻译，我们可以认为这两部分属于翻译IR到host指令，为了看清楚
 IR到host指令的翻译，我们首先要明确前端翻译得到的IR是怎么样的。

 IR指令的理解是比较直白的，qemu定义了一套IR的指令，具体的定义在tcg/README里说明,
 在一个tb里，qemu前端翻译得到的IR被串联到一个链表里，中间码优化和后端翻译都靠这个
 链表得到IR，中间码优化时，需要改动IR时(比如，删掉不可达的IR)，对这个链表做操作就
 好。

 中间码不只是定义了对应的指令，也有寄存器的定义，它形成了一个独立的逻辑空间，在IR
 这一层，可以认为都在中间码相关的寄存器上做计算的。IR这一层定义了几个寄存器类型，
 它们分别是：global, local temp, normal temp, fixed, const, ebb
```
 typedef enum TCGTempKind {                                                      
     /* Temp is dead at the end of all basic blocks. */                          
     TEMP_NORMAL,                                                                
     /* Temp is live across conditional branch, but dead otherwise. */           
     TEMP_EBB,                                                                   
     /* Temp is saved across basic blocks but dead at the end of TBs. */         
     TEMP_LOCAL,                                                                 
     /* Temp is saved across both basic blocks and translation blocks. */        
     TEMP_GLOBAL,                                                                
     /* Temp is in a fixed register. */                                          
     TEMP_FIXED,                                                                 
     /* Temp is a fixed constant. */                                             
     TEMP_CONST,                                                                 
 } TCGTempKind;                                                                  
```
 一般guest的gpr也被定义为IR这一层的global寄存器，中间码做计算的时候，会用到一些
 临时变量，这些临时变量就保存在local temp或者是normal temp这样的寄存器里，计算的
 时候要用到一些常量时，需要定义一个TCG寄存器，创建一个常量并把它赋给TCG寄存器。

 global、local temp、normal temp和const这些TCG寄存器我们在写前端翻译的时候会经常
 用到，fixed和ebb直接用到的情况不多。
 
 TCG寄存器是怎么定义和被使用的，以及它们本质上是什么？我们基于riscv看下这个问题。
 riscv下global寄存器一般如下定义：
```
 /* target/riscv/translate.c */
 riscv_translate_init
   [...]
   +-> cpu_gpr[i] = tcg_global_mem_new(cpu_env, 
   		    offsetof(CPURISCVState, gpr[i]), riscv_int_regnames[i]);            
         /* 在TCGContext里分配对应的空间，并且设定这个寄存器是TEMP_GLOBAL */
     +-> tcg_global_mem_new_internal(..., reg, offset, name);  
   [...]
   +-> cpu_pc = tcg_global_mem_new(cpu_env, offsetof(CPURISCVState, pc), "pc");    
   [...]
```
 我们只挑了gpr和pc的这几行代码出来，这里分配了对应的TCG寄存器，返回值是这些寄存器
 存储地址相对tcg_ctx的偏移。注意这里得到的是global寄存器的描述结构，类型是TCGTemp，
 而global寄存器实际存储在CPURISCVState内具体定义的地方，TCGTemp内通过mem_base和
 mem_offset指向具体存储地址。

 实际上，所有TCG寄存器的分配都是在TCGContext里分配了对应的存储空间，并且配置上相关
 参数，这些参数和IR一起交给后端做IR优化和后端翻译，后端使用TCGContext的地址和具体
 寄存器的偏移可以找见具体的TCG寄存器。

 local temp和normal temp的说明在[这里](https://wangzhou.github.io/qemu中basic-block以及tcg中各种变量的基本逻辑)有说明。简而言之，normal temp只在一个BB中有效，
 local temp在一个TB中有效。

 fixed要结合host寄存器分配来看，首先IR中分配的这些寄存器都是虚拟的寄存器，IR翻译
 到host指令都要给虚拟寄存器分配对应的host物理寄存器，当一个TCG寄存器有TEMP_FIXED
 标记表示在后端翻译时把这个虚拟寄存器固定映射到一个host物理寄存器上，一般fixed寄
 存器都是翻译执行时经常要用到的参数。
```
 /* tcg/tcg.c */
 tcg_context_init
   /*
    * 在tcg后端公共逻辑里，定义一个TCG寄存器，并把它绑定到host的TCG_AREG0这个寄存器上
    * 每个host都会把具体的实际物理寄存器映射到TCG_AREG0。
    */
   +-> ts = tcg_global_reg_new_internal(s, TCG_TYPE_PTR, TCG_AREG0, "env");        
   +-> cpu_env = temp_tcgv_ptr(ts);                                                
```
 如上的cpu_env依然是cpu_env寄存器存储地址针对tcg_ctx的偏移，前端翻译代码里会大量
 的用到cpu_env这个值，所以这里把它定义成fixed寄存器，提示后端翻译把cpu_env的值固定
 的放到一个host寄存器里。具体看，tcg_global_reg_new_internal里会把被绑定的host物理
 寄存器放到reserved_regs集合，这样，后端翻译后续就不会分配这个物理寄存器，cpu_env
 保存的是guest CPU软件结构体的指针，那么这个指针又是怎么传递给reserved TCG_AREG0
 host物理寄存器？可以看到一个tb执行的时候这个指针作为第一个入参传递给tb里生成的
 host指令：
```
 cpu_exec
   +-> cpu_loop_exec_tb
     +-> cpu_tb_exec(cpu, ...)
       +-> tcg_qemu_tb_exec(env, ...)
```
 在tb头里，会有一条指令把这个入参送给TCG_AREG0(ARM64的x19)，我们看看ARM64作为后端
 时，这个代码生成的逻辑：
```
 /* tcg/aarch64/tcg-target.c.inc */
 tcg_target_qemu_prologue
   [...]
   +-> tcg_set_frame(s, TCG_REG_SP, TCG_STATIC_CALL_ARGS_SIZE, CPU_TEMP_BUF_NLONGS * sizeof(long));
     +-> tcg_global_reg_new_internal(s, TCG_TYPE_PTR, reg, "_frame");          
   [...]
   +-> tcg_out_mov(s, TCG_TYPE_PTR, TCG_AREG0, tcg_target_call_iarg_regs[0]);      
```
 如上，公共代码里还会把host的栈寄存器也reserv出来做特定的用途，这里sp就是host上
 sp自己的语意，因为host调用翻译好的host指令，就是一个host上的函数调用。

 ebb类型的TCG寄存器表示这个寄存器可以跨越条件跳转，但是跨越之后状态为dead，这种
 变量类型和indirect的global寄存器有关系，在liveness_pass_2中会为indirect的global
 寄存器新创建ebb类型的TCG寄存器，具体用法还待分析。

 (todo: 具体用法)
 
中间码优化
-----------

 前端翻译得到的IR可能会有优化的空间存在，所以qemu在进行后端翻译之前会先做中间码
 优化，优化以一个TB为单位，优化的输入就是一个TB对应的IR和用到的TCG寄存器。
```
 /* tcg/tcg.c */
 tcg_gen_code
   +-> tcg_optimize(s)
     +-> done = fold_add(&ctx, op);                                          

   +-> reachable_code_pass(s);                                                     
```
 tcg_optimize是做着一些常量的检查，进而做指令优化(折叠常量表达式), 我们取其中的
 一个case，比如fold_add具体看下，大概知道下这里是在干什么。可以看到这个case检测
 add_32/64这个IR的两个操作数是不是常量，如果是常量，那么在这里直接把常量相加后的
 结果放到一个常量类型TCG寄存器，然后把之前的add_32/64改成一条mov指令。

 从名字就可以看出reachable_code_pass应该做的是一些死代码的删除，这里检测到运行不到
 的IR就直接从IR链表里把他们删掉。

 中间码优化的输出还是IR链表和相关的TCG寄存器，可见我们也可以把这两个函数注释掉，
 从而把中间码优化关掉。可以看出，中间码优化和编译器IR优化的逻辑是类似的。

 中间码优化的具体case本文就不继续展开了，后续有需要再写吧。

寄存器活性分析
---------------

 qemu最终还是要把IR和TCG寄存器翻译成host指令和host寄存器，才能在host机器上运行,
 这一节和下一节就是要解决这个问题。直观来看，IR和host指令大概是可以对应上的，这里
 要解决的关键问题就是怎么把虚拟机的TCG寄存器映射到host物理寄存器上。

 我们先看下具体的两条riscv指令是怎么翻译成host指令的:
```
riscv guest汇编:

  0x0000000000010172:  1101              addi            sp,sp,-32
  0x0000000000010174:  ec06              sd              ra,24(sp)

中间码：

  ---- 0000000000010172 0000000000000000
  add_i64 x2/sp,x2/sp,$0xffffffffffffffe0
  
  ---- 0000000000010174 0000000000000000
  add_i64 tmp4,x2/sp,$0x18
  qemu_st_i64 x1/ra,tmp4,leq,0

ARM64 host汇编：

    -- guest addr 0x0000000000010172 + tb prologue
  0xffff9c000140:  b85f8274  ldur     w20, [x19, #-8]
  0xffff9c000144:  7100029f  cmp      w20, #0
  0xffff9c000148:  5400064b  b.lt     #0xffff9c000210
  0xffff9c00014c:  f9400a74  ldr      x20, [x19, #0x10]
  0xffff9c000150:  d1008294  sub      x20, x20, #0x20
  0xffff9c000154:  f9000a74  str      x20, [x19, #0x10]
    -- guest addr 0x0000000000010174
  0xffff9c000158:  91006295  add      x21, x20, #0x18
  0xffff9c00015c:  f9400676  ldr      x22, [x19, #8]
  0xffff9c000160:  f83f6ab6  str      x22, [x21, xzr]
```
 riscv的addi被翻译成中间码add_i64, 注意中间码中的x2/sp是TEMP_GLOBAL类型的TCG寄存器，
 riscv的sd指令被翻译成两条中间码，第一个中间码计算store的地址，并存在tmp4里，第二个
 中间码把ra寄存器的值保存到tmp4指向的地址。

 我们看下实际翻译出来的ARM64指令，第一条指令合并了一点tb头的指令，addi对应的host
 指令是从0xffff9c00014c这里开始的，从上面知道x19就是cpu_env的指针，0x10是riscv sp
 对应的TCG寄存器在cpu_env的偏移，所以“ldr x20, [x19, #0x10]”就是把保存在内存里的
 guest CPU sp的值load到host寄存器x20上，下面sub指令对应的就是riscv的addi指令，然后
 紧接着一个str指令把sp的值更新回cpu_env，注意x20还是sp的值，所以, host还是可以使用
 x20中保存的sp计算指令“sd ra,24(sp)”中sd要保存值的地址，翻译到host上的“add x21, x20, #0x18”，
 x21保存sd要保存值的地址，后面的“ldr x22, [x19, #8]”同样把riscv的ra load到host寄存器
 x22上，最后host使用“str x22, [x21, xzr]”完成“sd ra,24(sp)”的模拟。

 需要注意的是，如上的log是用qemu的user mode下得到的，user mode没有地址翻译，所以
 store的模拟才会如此直接。如上的host寄存器对应的TCG寄存器类型: x19是fixed，x20/x22
 是global，x21是temp。寄存器如何分配、分配出的host寄存器什么时候可以重复利用、host
 寄存器上的值什么时候需要保存回cpu_env，这些都是活性分析和后端翻译要考虑的问题。

 寄存器活性分析代码主体逻辑如下:
```
 tcg_gen_code
   +-> liveness_pass_1(s);                                                         
   /*
    * nb_indirects的值在创建global TCG寄存器的时候更新: tcg_global_mem_new_internal，
    * 这个函数会检测base入参的TCG类型，注意不是自己的TCG类型，如果base的类型是global
    * 才会增加nb_indirects的计数。一般调用这个函数为guest gpr创建global TCG寄存器
    * 都是用cpu_env作为base入参，所以nb_indirects的值都不会增加。
    *
    * 也就是qemu认为，对于global虚拟寄存器的访问，如果是通过一个fix寄存器作为指针
    * 访问，就叫direct，但是如果不是，就叫indirect。针对indirect的访问需要进行额外
    * 的liveness_pass_2优化。
    *
    * 目前还有没有想到需要liveness_pass_2优化的例子。
    */
   +-> if (s->nb_indirects > 0) {
           if (liveness_pass_2(s)) {                                               
               liveness_pass_1(s);                                                 
           }                                                                       
       }
```

 如上，我们目前先分析liveness_pass_1的逻辑，IR和TCG寄存器的数据结构大概是这样的：
```
 tcg_ctx:
         +--------+---------+---------+---------+---------+
         | temp0  |  temp1  |  temp2  |  temp3  |  temp4  |
         +--------+---------+---------+---------+---------+
            ^        ^                   ^           ^
 TB ops:    |        +----------------+  |           |
            +-------------+-------+   |  +--------+  |                 
                          |    +--+---+-----------+--+                ^
                          |    |  |   |           |                   |
         +----------------+----+--+---+-----------+----------------+  |
         | +-----+      +-+--+ |  | +-+--+      +-+--+             |  |
         | |insn0|      |arg0| |  | |arg1|      |arg2|        life |  |  parse insn
         | +-----+      +----+ |  | +----+      +----+             |  |
         +---------------------+--+--------------------------------+  |
         +---------------------+--+--------------------------------+  |
         | +-----+      +----+ |  | +----+                         |  |
         | |insn1|      |arg0+-+  +-+arg1|                    life |  |
         | +-----+      +----+      +----+                         |  |
         +---------------------------------------------------------+  |
           ...                                                        |
```
 如上所示，前端翻译生成的IR组成一个IR链表，每个IR节点里有它自己的寄存器定义和life，
 这个life标记当前IR中每个寄存器的状态。IR中的每个TCG变量指向tcg_ctx中TCG变量的实际
 保存地址，活性分析对于TB中的IR，按照**逆序**逐个分析对应的IR和IR的TCG寄存器的状态，
 分析过程把TCG寄存器的状态动态的更新到tcg_ctx的TCG寄存器对象中，位置相对在上面的
 IR的TCG寄存器状态受下面IR的TCG寄存器状态的影响，而下面的TCG寄存器状态在分析的时候
 已经更新到tcg_ctx的TCG寄存器对象中，每条IR的寄存器分析完后的静态状态保存在op->life
 里。

 TCG寄存器的状态有两种，分别是TS_DEAD和TS_MEM，TS_DEAD的寄存器表示这个寄存器不会
 被后续(顺序)的指令使用，TS_MEM的表示寄存器需要向内存同步。

 TCG寄存器在遍历开始的初始值是：global变量是TS_DEAD | TS_MEM, 其它是TS_DEAD。所有
 global变量，比如gpr，都要刷回内存，其它的变量都是临时变量(cpu_env，sp也不需要刷会
 内存)，先都配置成dead，如果后续检测到寄存器之间存在依赖，再配置成live(我们把dead
 这个状态被去掉，认为TCG寄存器变成live状态)。
```
 IR0: add temp2, temp4, temp5
 IR1: add temp1, temp2, temp3
```
 用如上的两个IR举个例子，qemu从IR1开始分析，IR1的temp1/temp2/temp3都会配置成dead，
 因为后续没有指令，也就没有指令会使用，分析IR0时，qemu会检测到temp2在IR1里会使用，
 就会把IR0对应的temp2配置成live。可以看到寄存器dead/live的状态是针对具体指令的，
 寄存器dead/live状态会直接影响到后续host寄存器的分配，分配寄存器的时候，当一个虚拟
 寄存器是dead时，它后续不会被用，qemu就可以把这个虚拟寄存器对应的host寄存器重新分配
 使用，反之不行。

 liveness_pass_1的逻辑大概是这样的：
```
 liveness_pass_1
   /* 遍历开始，更新TCG寄存器为初始状态 */
   la_func_end(s, nb_globals, nb_temps);                                       
   /*
    * 逆序遍历TB的中间码链表，除了几种类型的中间码要特殊处理下，剩余的都在默认
    * 处理分支里(default)。需要单独处理的中间码有：call、insn_start、discard、
    * 多输出的中间码(add2/sub2/mulu2/muls2_i32/i64)。
    *
    * 我们先关注default流程，然后再看需要单独处理的中间码。
    */
   QTAILQ_FOREACH_REVERSE_SAFE(op, &s->ops, link, op_prev)
   /* 
    * 如下是switch中的default的逻辑。
    *
    * 对于不是side_effect的指令，只要有输出参数不是dead，就不能去掉这条指令，否则
    * 所有输出参数都dead了，这个指令就可以去掉了。
    */
   do_remove:
     +-> tcg_op_remove(s, op);                                               

   /* 寄存器活性分析核心逻辑在这里 */
   do_not_remove:
     /*
      * 首先处理IR的输出寄存器，根据TCG寄存器状态更新IR的life，更新完后把TCG寄存
      * 器状态配置为dead，对于输出寄存器，必然不会对之前IR的寄存器有依赖。
      */
     for (i = 0; i < nb_oargs; i++) {
         ts = arg_temp(op->args[i]);

         op->output_pref[i] = *la_temp_pref(ts);

         /* Output args are dead. */
         if (ts->state & TS_DEAD) {
             arg_life |= DEAD_ARG << i;
         }

         if (ts->state & TS_MEM) {
             arg_life |= SYNC_ARG << i;
         }
	 /*
	  * 这里对所有输出都配置dead，是因为显然当前这个寄存器在程序运行时不会依
	  * 赖前序指令中的这个寄存器。
	  */
         ts->state = TS_DEAD;
         la_reset_pref(ts);
     }

     /*
      * 处理TB结束、BB结束、条件跳转以及有side effect的指令。TCG_OPF_BB_EXIT是
      * 离开TB，所以temp dead，global dead和sync。
      */
     if (def->flags & TCG_OPF_BB_EXIT) {
         la_func_end(s, nb_globals, nb_temps);
     /*
      * 把条件branch指令单拿出来看，条件branch指令只是结束一个BB的一种情况, 结束
      * 一个BB还有goto_tb和exit_tb，开始一个BB还有set_label。     
      *                                                               
      * insn5                         |  BB0                          
      * insn6                         |                               
      * brcond t0, t1, cond, label  --+                               
      * insn1                       --+                               
      * insn2                         |  BB1                          
      * insn3                         |                               
      * insn4     --------------------+                               
      * set_label --------------------+                               
      * insn7                         |  BB2                          
      * insn8                         |                               
      *                                                               
      * 从下到上解析到brcond的时候，所有global和local要sync，但是不一定dead, 普通
      * temp要都dead, 也就是说在程序执行时，后续指令不会再使用temp寄存器，这个和
      * qemu规定的normal temp不能垮BB使用是一致的。对于所有的EBB和const，TCG寄存
      * 器状态不改变。
      */                                                                 
     } else if (def->flags & TCG_OPF_COND_BRANCH) {
         la_bb_sync(s, nb_globals, nb_temps);
     /*
      * BB_END时，也就是br(直接跳转)、goto_tb、exit_tb以及set_label之前，在上面
      * brcond的基础上EBB/const要dead，fixed要sync。但是原因是？
      */
     } else if (def->flags & TCG_OPF_BB_END) {
         la_bb_end(s, nb_globals, nb_temps);
     /*
      * 看起来只有ld/store是有effect的指令，根据qemu注释，load/store可能触发异常，
      * global寄存器作为guest CPU的上下文信息，要刷回表示表示guest CPU的内存中。
      */
     } else if (def->flags & TCG_OPF_SIDE_EFFECTS) {
         la_global_sync(s, nb_globals);
         /* 这里没有搞明白？*/
         if (def->flags & TCG_OPF_CALL_CLOBBER) {
             la_cross_call(s, nb_temps);
         }
     }

     /*
      * 处理输入寄存器状态。对于输入寄存器，如果之前已经dead，对于本条指令，这个
      * 寄存器是dead，因为后面没有人用了，所以配置当前IR的life。但是再往上遍历，
      * 因为这个寄存器在这里使用了，就要激活TCG寄存器，这个就是下面一段代码做的事。
      */
     for (i = nb_oargs; i < nb_oargs + nb_iargs; i++) {                  
         ts = arg_temp(op->args[i]);                                     
         if (ts->state & TS_DEAD) {                                      
             arg_life |= DEAD_ARG << i;                                  
         }                                                               
     }                                                                   

     /*
      * 激活输入TCG寄存器。当有一条指令的input用了一个寄存器，那么这个寄存器当然
      * 要live了，这里配置成live，是给产生这个input的指令看的，当后续逆序解析到
      * 这条指令的时候，对应的寄存器就不能是dead。
      */
     for (i = nb_oargs; i < nb_oargs + nb_iargs; i++) {                  
         ts = arg_temp(op->args[i]);                                     
         if (ts->state & TS_DEAD) {                                      
             /* 得到可以使用的host寄存器的集合 */
             *la_temp_pref(ts) = tcg_target_available_regs[ts->type];    
             ts->state &= ~TS_DEAD;                                      
         }                                                               
     }                                                                   

     /* todo: 寄存器传递？*/
```

下面是单独case处理的中间码的相关TCG寄存器的分析，insn_start直接跳过，因为insn_start
只是一个hint，discard表示这个指令标记的寄存器后面没有再使用了，所以直接配置对应
的TCG寄存器为dead，剩下是call和一堆二输出的中间码。

二输出中间码和对应的单输出的中间码的逻辑是一致的，只不过每个输入输出值是由两个TCG
寄存器组成，一个存放低32或64bit，一个存放高32或64bit。二输出的中间码又分为加减和
乘法两类，如果两个输出TCG都是dead，对应的IR可以删除，二输出的加减IR，如果只有高位
输出寄存器是dead，IR可以转化成单输出加减IR，二输出的乘法IR，输出其中之一是dead时，
IR可以分别转化为不同的单输出乘法IR(todo: 没有搞清这里的逻辑)。

call中间码用来支持helper函数，对于没有副作用的函数，如果输出都dead, 就可以删去掉
这个call中间码，也就没有对应的helper函数调用了。call的一般处理逻辑和上面的逻辑是
基本一致的，从IR的角度看，call可以看成是一条用户自定义的顺序执行的IR，并不会改变
BB的划分，另外qemu提供了一组call_flag，程序员可以用call_flag描述helper函数的一些
共同特征，比如有没有副作用、有没有对global寄存器的读写等，在处理call相关的寄存器
时，qemu根据这些call_flag做寄存器的同步。
```
 case INDEX_op_call:
   /* 处理没有副作用的情况，这里主要是判断能不能删掉call */
   if (call_flags & TCG_CALL_NO_SIDE_EFFECTS)
       [...]

   /* 处理输出寄存器和上面的分析基本一致，但是会把op->output_pref[i]清0 */
   [...]
   op->output_pref[i] = 0;

   /*
    * 处理call_flag相关的global寄存器同步，helper函数里有对global寄存器的读写。
    * kill和下面sync的区别是，kill是sync+dead。有对global的写，为啥要dead global? 
    * 这里和一个IR的输出是一样的逻辑，被写的global不可能是：在上面一条IR的输出，
    * 然后这里的输入。
    */
   if (!(call_flags & (TCG_CALL_NO_WRITE_GLOBALS | TCG_CALL_NO_READ_GLOBALS)))
       la_global_kill(s, nb_globals);
   /*
    * helper函数里有对全局变量的读。因为helper函数有直接读global(读cpu_env上的变量), 
    * 所以调用之前必须把host寄存器上的值刷回内存。qemu这里是全部刷回内存了，其实
    * 是没有必要的，只刷回helper函数里要用的就可以，但是这个信息比较难拿到。所以，
    * 后续的后端翻译，要在call之前，要先处理对应的寄存器，而且其它的IR也要这样。
    */
   else if (!(call_flags & TCG_CALL_NO_READ_GLOBALS))
       la_global_sync(s, nb_globals);

   /* 这里没有搞懂？*/
   la_cross_call(s, nb_temps);

   /*
    * 处理输入寄存器和上面的分析基本一致，state_ptr上略有不同，call的输入参数可能
    * 会很多，这里只能对host参数传递寄存器个数范围内的虚拟寄存器指定state_ptr。
    * 另外特殊的地方是，call会直接分配输入寄存器。
    */
   *la_temp_pref(ts) = (i < nb_call_regs ? 0 : tcg_target_available_regs[ts->type]);
   tcg_regset_set_reg(*la_temp_pref(ts), tcg_target_call_iarg_regs[i]);
```

后端翻译
---------

 进入后端翻译的主流程前在tcg_reg_alloc_start函数中先根据TCG寄存器类型(kind)得到
 TCG寄存器的val_type域段的值，这个域段是一个动态值，指示的是TCG寄存器值对应的存储
 状态，比如TEMP_VAL_REG表示当前TCG寄存器的值保存在host寄存器上，TEMP_VAL_MEM表示
 当前TCG寄存器的值保存在内存里(cpu_env的TCG寄存器对应域段)，TEMP_VAL_CONST表示常量，
 TEMP_VAL_DEAD表示一个寄存器不需要从TCG load到host寄存器使用。所以，具体的映射初始
 值是：fixed -> TEMP_VAL_REG，global/local temp -> TEMP_VAL_MEM，const -> TEMP_VAL_CONST，
 normal temp/ebb -> TEMP_VAL_DEAD。

 正序遍历TB的IR链表，逐个翻译每个中间码和TCG寄存器，这个是后端翻译的主流程。可以
 看到，这里针对几个特殊的中间码做特殊处理，主流程在tcg_reg_alloc_op里。
```
 QTAILQ_FOREACH(op, &s->ops, link) {
     case ...
     default:
         tcg_reg_alloc_op(s, op);
 }
```

 分配过程会涉及到IR定义里的args_ct域段，这个域段描述IR翻译成特定host指令的限制，
 我们先看IR定义里的args_ct如何初始化，明确其中的含义。IR指令的定义在qemu公共代码
 里初始化:
```
 /* tcg/tcg-common.c */
 TCGOpDef tcg_op_defs[] = {
 #define DEF(s, oargs, iargs, cargs, flags) \
          { #s, oargs, iargs, cargs, iargs + oargs + cargs, flags },
 #include "tcg/tcg-opc.h"
 #undef DEF
 };
```
 其中的参数分别是指令名字、输出参数个数、输入参数个数、指令控制参数个数(比如brcond
 里的cond)、指令flag(描述指令附加的一些属性)，注意这里只静态定义了每个IR的公共部分，
 并没有定义args_ct，args_ct和host指令的特点有关系，所以自然定义在具体host代码里。
 args_ct初始化的代码路径是：
```
 tcg_context_init
       /* 遍历每个IR，得到host定义的针对每个IR的约束的定义 */
   +-> process_op_defs(s);
         /*
          * 如果host是riscv，tcg_target_op_def就是定义在tcg/riscv/tcg-target.c.inc，
          * 可以看到con_set的值是一个枚举值，对应枚举元素的定义类似：c_o1_i2_r_r_rI
          * 这个枚举类型定义在tcg/tcg.c，枚举元素include host上的具体定义：
          * 
          * typedef enum {
          * #include "tcg-target-con-set.h"  <- 如果rv是host，就是tcg/riscv/tcg-target-con-set.h
          * } TCGConstraintSetIndex;
          *
          * 继续从constraint_sets得到这个宏对应的字符串，这里重定义了名字相同的参数
          * 宏，使得名字相同的宏对应的代码不一样：
          * 
          * static const TCGTargetOpDef constraint_sets[] = {
          * #include "tcg-target-con-set.h"
          * };
          *
          * 比如，还是如上的枚举元素，这里得到的tdefs包含一个字符串数组，其中的
          * 每个字符串是：r，r，rI，一个参数可能有多个属性的叠加，比如这里的最后
          * 一个参数就有r和I。
          *
          * 这代码写的也是风骚！具体字符的解析下面分析。
          */
     +-> con_set = tcg_target_op_def(op);
     +-> tdefs = &constraint_sets[con_set];
           /* 解析一个IR中每个输入输出参数的限制，更新到args_ct域段 */
       +-> for (i = 0; i < nb_args; i++) {
               while (*ct_str != '\0') {
               /* 数字的含义没有看懂，似乎表示alias */
               case '0' ... '9'
               /* 很少用到，是需要分一个新寄存器的意思？*/
               case '&'
               /* 表示需要一个常数，但是rv上是i */
               case 'i':
               }
               /*
                * 特定host还可以自定寄存器的限制塞到这里，比如，rv在这里塞了如下
                * 的case。从这里也可以看出，args_ct里reg表示寄存器的约束，ct表示
                * 常量的约束。
                *
                * 特定host的约束在host代码里具体定义，比如，下面的ALL_GENERAL_REGS
                * 就定义在tcg/riscv/tcg-target.c.inc，是MAKE_64BIT_MASK(0, 32)
                *
                * qemu里分配寄存器的公共代码的入参中就包含了这里定义的具体约束。
                */
               case 'r': def->args_ct[i].regs |= ALL_GENERAL_REGS; ct_str++; break;
               ...
               case 'I': def->args_ct[i].ct |= TCG_CT_CONST_S12; ct_str++; break;
               ...
           }
           /* 根据特定优先级分配为输出输入参数排序 */
       +-> sort_constraints(def, 0, def->nb_oargs);
           sort_constraints(def, def->nb_oargs, def->nb_iargs);
```
 
 单个IR生成host指令以及分配host寄存器的过程：
```
 tcg_reg_alloc_op(s, op)
       /*
        * 整个翻到host指令的过程，关键是分配寄存器，在一个TB里分配寄存器，那么就
        * 要有中间变量把分配的和还没有分配的寄存器记录下来。
        *
        * reserved_regs表示被保留起来的host上寄存器，TB块里不能用, 所以这里直接
        * 标记为已分配。注意对于reserved_regs这里有一个独立的逻辑，拿riscv后端举
        * 例，除了sp/tp/gp/zero寄存器，riscv还会把reserve出几个gpr临时使用，公共
        * 的寄存器分配已经基本完成了host物理寄存器的分配，但是在具体创建host指令
        * 的时候有的IR可能还需要插入新host指令，这样就需要分配新的host物理寄存器，
        * 提前reserve出的host gpr就是用来做这个。
        *
        * 这个函数只是翻译一个IR，几个核心的数据结构的含义是：new_args[]、con_args[]
        * 指的是当前这个IR翻译到host指令时，分配得到的host物理寄存器。s中的reserved_regs、
        * reg_to_temp[]指的是tb翻译上下文host寄存器的分配情况，所以，当一个host
        * 物理寄存器被分配或释放时需要更新s中的reg_to_temp[]对应元素。
        */
   +-> i_allocated_regs = s->reserved_regs;
       o_allocated_regs = s->reserved_regs;

       /* 处理输入参数 */
   +-> for (k = 0; k < nb_iargs; k++) {
       /*
        * 处理输入参数限制, 参数优先级排序还不清楚? 如果是常数，并且根据host
        * 指令可以表示常量的限制，判断是否可以直接把常量编码到指令里，如果可以，
        * 就把信息记录在const_args和new_args，这两者都是后续生成host指令的参数。
        * 如果无法编码到指令里，就要分配寄存器，使用寄存器保存参数，参与计算。
        *
        * 看一个实际的例子，host是riscv时，翻译add_i64这个中间码。add_i64的两个
        * 输入参数，如果后一个参数是立即数，就有可能被翻译成riscv上的addi，否则
        * 就需要翻译成add，但是addi的立即数只有12bit位宽，如果装不下的话，还是需要
        * 把立即数搬到host寄存器上，翻译成add指令。参照如上分析，这个立即数位宽
        * 的限制被生成到add_i64中间码的arg_ct的ct域段，具体在riscv tcg_target_op_def
        * case INDEX_op_add_i64的C_O1_I2(r, r, rI)，rI表示add_i64翻译到riscv上时，
        * 最后一个参数可以被翻译到host寄存器或者立即数编码到指令。当qemu前端翻译
        * 使用add_i64这个中间码并且第二个入参定义成CONST时，如下的代码会得到这个
        * 信息并使用tcg_target_const_match做检测，如果检查成功就做个标记，同时结束
        * 当前寄存器的分配，后面就会根据这个标记生成addi，把第二个虚拟寄存器直接
        * 编码到host指令里。如果检测不成功，就继续分配host物理寄存器，所以可见const
        * 虚拟寄存器可能直接编码到指令或者占用一个host物理寄存器。
        */
       if (ts->val_type == TEMP_VAL_CONST
           && tcg_target_const_match(ts->val, ts->type, arg_ct->ct)) {
           const_args[i] = 1;
           new_args[i] = ts->val;
       }

       /* ialias这个没有看懂？*/
       if (arg_ct->ialias) {}

       /*
        * 给输入参数分配host寄存器，并且从cpu_env中load输入参数到host寄存器，为后
        * 续计算做准备。arg_ct->regs是host指令在host寄存器分配上的限制。
        *
        * 从这个函数的逻辑就可以看出TCG寄存器val_type的语意，它表示TCG寄存器当前
        * 的存储状态。TEMP_VAL_REG表示已经在host寄存器里，所以直接返回。
        *
        * TEMP_VAL_CONST表示是一个常量(并且当前保存在TCG寄存器)，这里就要分配一个
        * host寄存器，并且生成一条host movi指令把这个常量送到host寄存器上，并且
        * 配置ts->mem_coherent = 0，这个表示TCG寄存器和host寄存器不同步。
        *
        * TEMP_VAL_MEM表示TCG寄存器的值在内存里，这里要分配host寄存器，并把对应
        * 的值load进host寄存器，同时配置ts->mem_coherent = 1。从host分配物理寄存器
        * 就会有分不到的情况，遇到这种情况就需要把host寄存器先换到内存，并且标记
        * 这个虚拟寄存器的值在内存上，tcg_reg_free完成这个动作。
        * 
        * TEMP_VAL_DEAD表示一个寄存器不需要从TCG load到host寄存器使用。tcg_reg_alloc_start
        * 把TEMP_NORMAL/TEMP_EBB转换成TEMP_VAL_DEAD，像normal temp和ebb这种中间
        * 计算产生的数据，显然始终产生于一个左值，用于存放临时变量(生命周期只在
        * 一个BB内)，不需要刷回内存，更不需要从没存load进host寄存器。
        */
       temp_load(s, ts, arg_ct->regs, i_allocated_regs, i_preferred_regs);
             /* 如上分析，处理各种val_type的情况 */
         +-> switch (ts->val_type) {}

             /*
              * 返回用掉的寄存器，注意这里一定会分到host物理寄存器，没有时也会像
              * 上面说的那样换一个host物理寄存器出来。
              */
         +-> ts->reg = reg;
             /* TEMP_VAL_REG表示这个虚拟机寄存器的值已经在host寄存器里了 */
         +-> ts->val_type = TEMP_VAL_REG;
             /* 表示当前物理寄存器对应的虚拟寄存器 */
         +-> s->reg_to_temp[reg] = ts;
       }

       /*
        * 处理dead输入寄存器。上面是考虑如何往出分配host物理寄存器，这里是考虑怎
        * 么回收host物理寄存器。只需要考虑已经dead的寄存器，就是后续不会再用的寄
        * 存器，对于global、local temp，要刷回cpu_env内存，对于normal temp、ebb
        * 只需要dead就好，它们本来就是临时变量。
        *
        * 需要注意的是，temp_free_or_dead这里使用的是dead，也就是对于normal temp
        * 和ebb是dead，另外针对normal temp和ebb的free操作体现在host寄存器不够用的
        * 时候，qemu可以把这些临时变量也换到内存里。(这些临时变量何时销毁？)
        *
        * 这里并没有插入刷到内存的具体操作，只是更新虚拟寄存器的存储状态(val_type)，
        * 以及tb翻译上下文中host物理寄存器的翻译状态(s->reg_to_temp): 如果当前已
        * 经分配物理寄存器，但是dead了，那么这个物理寄存器就可以给其它虚拟寄存器
        * 用，所以，清理掉reg_to_temp中reg到temp的指向。
        */
   +-> for (i = nb_oargs; i < nb_oargs + nb_iargs; i++) {
         if (IS_DEAD_ARG(i)) {
           temp_dead(s, arg_temp(op->args[i]));
             temp_free_or_head()
             [...]
         }
       }

       /*
        * 检查条件跳转、BB结尾以及side effect的情况, 处理call_clobber。检查只是做
        * assert，没有逻辑处理。
        */
   +-> if (def->flags & TCG_OPF_COND_BRANCH)
         [...]
       else if (def->flags & TCG_OPF_BB_END)
         [...]
       else {
         /*
          * 已经有分支单独处理call，这里就只处理st/ld了，这里会free clobber寄存器,
          * 原因是什么？
          */
         if (def->flags & TCG_OPF_CALL_CLOBBER)
           [...]
         /* 只是做检查，没有真实sync。确保内存里记录的虚拟寄存器值是对的 */
         if (def->flags & TCG_OPF_SIDE_EFFECTS)
           [...]

         /* 处理输出参数 */
   +---> for(k = 0; k < nb_oargs; k++) {
           /* arg_ct这段没有看懂？如下是最后一个分支 */
           reg = tcg_reg_alloc(s, arg_ct->regs, o_allocated_regs,
                               op->output_pref[k], ts->indirect_base);

           tcg_regset_set_reg(o_allocated_regs, reg);
           /* 没有理解这里？*/
           if (ts->val_type == TEMP_VAL_REG) {
               s->reg_to_temp[ts->reg] = NULL;
           }

           /* 分了一个物理寄存器，所以这个虚拟寄存器的值现在保存在物理寄存器上 */
           ts->val_type = TEMP_VAL_REG;
           ts->reg = reg;

           /* TCG寄存器对应的物理寄存器和内存值当前不一致性 */
           ts->mem_coherent = 0;
           s->reg_to_temp[reg] = ts;
           new_args[i] = reg;
         }
       }

       /* 根据参数构建指令，各个不同host实现自己的回调 */
   +-> tcg_out_op(s, op->opc, new_args, const_args);

       /* 把global输出寄存器刷回cpu_env */
   +-> for(i = 0; i < nb_oargs; i++) {
         /*
          * 核心是处理sync，实现寄存器活性分析里已经确定的需要sync的寄存器，TEMP_VAL_REG
          * 即当前值在host寄存器上时，才要刷回内存。为什么TEMP_VAL_CONST有时要先load在store？
          */
         if (NEED_SYNC_ARG(i))
             temp_sync(s, ts, o_allocated_regs, 0, IS_DEAD_ARG(i));

         /* 处理只dead的输出寄存器，只处理寄存器分配层面的逻辑 */
         else if (IS_DEAD_ARG(i))
             temp_dead(s, ts);
       }
```

下面看剩余IR的翻译，主要是mov/call/set_label。

qemu实现set_label的思路是，先把label记录在TCGContext的labels链表里，后端翻译br/brcond
时，把跳转指令的地址和labels建立联系，把跳到一个label的跳转指令地址都记录在label
的relocs链表里，注意一个label可能从不同的地方跳进来，在tcg_gen_code的结尾处调用
tcg_resolve_relocsg更新所有label对应的跳转指令的目的地址。

call其实就是要在TB function上下文里实现helper函数的调用。可以想象qemu的后端翻译
要做的有：helper函数入参准备，这个包括入参寄存器不够用的时候，用栈传递函数入参，
保存caller save寄存器，跳转到helper函数执行，从栈内恢复caller save寄存器。从qemu
具体的实现上看，caller save寄存器并没有被保存到栈上，而至直接刷回了对应cpu_env里，
这样只要改变下寄存器存储位置的标识就好，也不需要恢复caller save寄存器。
