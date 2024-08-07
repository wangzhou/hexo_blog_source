---
title: 现代CPU性能分析与优化-笔记
tags:
  - 计算机体系结构
description: >-
  本文是performance analysis and tuning on modern CPUs的读书笔记，原文的位置
  在https://book.easyperf.net/perf_book。这本书已经有了中文翻译的版本，总的
  看来翻译的指令还好，这本书基于CPU微架构讲了用perf调优的基本方法，perf也主要 是使用了Intel CPU
  PMU的相关event。总体上看，这本书虽然写的比较科普，但是对 调优时可能遇到的微架构概念都有清晰的介绍，值得一读。 这本书没有长逻辑链，
  所以我们就只是分章节单点记录下要点。
categories: read
abbrlink: 16222
date: 2023-12-28 19:38:12
---

第2章
------

性能测试时要注意CPU的动态调频，这个特性会给测量带来不确定性。(Linux系统下，动态
调频的控制一般在/sys/devices/system/cpu/cpux/cpufreq/下，默认是schedutil，大概的
意思是如果CPU负载重就提高频率跑，如果负载轻就可以适当降低频率跑。performance为
性能优先，会用最高频率跑)

记时有两种方法，一个是记录cpu cycle，一个是通过OS的clock_gettime记录实际的时间。
(Intel上在用户态可以通过指令查看cpu cycle，ARM还没有这样的功能)

第3章
------

Intel安腾构架(Itanium)使用超长指令字(VLIW)的设计。

硬件多线程机制主要的收益点是CPU存在后端长时延阻塞时(latency bound)，更加充分的利
用core资源。(注意，latency bound和memory bound的区别?)

第4章
------

这一章基于perf工具解释了一些关键指标的含义，下面直接在一台X86笔记本上使用perf看下，
值得一提的是，这个一台2017年的Intel笔记本，基本上的事件、topdown功能都直接支持了。
```
sherlock@x86:~/notes/perf$ sudo perf stat -a ls
[...]

 Performance counter stats for 'system wide': 

  3.463590   cpu-clock (msec)   #    3.723 CPUs utilized   <- cpu利用率, 非idle线程都计入
        13   context-switches   #    0.004 M/sec                  
         5   cpu-migrations     #    0.001 M/sec                  
       100   page-faults        #    0.029 M/sec                  
 4,489,365   cycles             #    1.296 GHz                    
 2,329,591   instructions       #    0.52  insn per cycle  <- IPC
   468,854   branches           #  135.366 M/sec                  
    18,710   branch-misses      #    3.99% of all branches <- 分支预测错误率

 0.000930215 seconds time elapsed
```

第5章
------

PMU采样模式会打断被采样进程执行，所以是有开销的(生产环境无法使用)。

perf record记录的热点分布上难以看到热点函数的调用者，比如热点可能是一个库函数这样
很多地方都可能调用到。perf record --call-graph <fp|dwarf|lbr>可以看到调用者的比例，
但是这三种方法对被测程序有一定的要求，fp要开--fnoomit-frame-pointer编译参数，dwarf
要加-g编译参数，lbr要硬件支持LBR(last branch record)这个特性，且跟踪调用深度有限。

屋顶线性能模型没有看懂。

编译器还提供了各种静态性能分析工具和编译器优化报告。

第6章
------

这个章节介绍了Intel CPU上一些高级的PMU特性，主要包括topdown分析方法、LBR(last
branch record)、PEBS(Processor Event-Based Sampling)以及PT(Processor Trace)。

在ARM上topdown的分析方法可以增加厂商自定义的PMU寄存器实现，PEBS应该对应的是SPE，
PT应该对应的Branch Record Buffer Extension，LBR在ARM上应该也可以用BRBE搞定。

topdown的分析方法Intel很早就在产品里使用，最早的对外呈现在论文"A Top-Down Method
for Performance Analysis and Counters Architecture"中，这本书很多内容就是论文里
的内容，可以认为是论文内容的简述。topdown方式目的是要层次化的得到CPU执行一个程序，
资源到底都花费在了哪里，从高层到低层逐步发现性能瓶颈点。其具体达成的手段是增加必要
的PMU counter，收集程序运行时PMU counter的数据，然后通过一定的算法得到性能瓶颈的
表述。 

topdown分析的最顶层是四类大的事件：front-end bound, back-end bound, bad speculation
和retiring。划分的方法是，对于一个可以被发射出去的uop，那么这个uop肯定是被对应的
执行单元执行了，所以它的结果只能有两个，一个是执行完成并且retire了，可以看到这个
其实就是真正做有用功的uop，另外一个是，这个uop处于错误预测路径上，最后被flush掉，
前者就叫retiring，后者就叫bad speculation。对于一个没有被发射出去的uop，也是有两种
情况，一个是后端的执行单元都忙着，虽然指令已经完成decode/rename等操作，但是后端
还没法执行，这个叫back-end bound, 另一个是，指令压根还没有做好发给后端的准备，这
个叫front-end bound。

Intel的笔记本上使用perf可以很简单的得到topdown分析的输出(ubuntu 18.04):
```
sherlock@x86:~/notes/perf$ sudo perf stat -a --topdown ls
[...]

 Performance counter stats for 'system wide':

            retiring       bad speculation   frontend bound    backend bound        
S0-C0     2     18.6%          9.0%            29.2%            43.3%           
S0-C1     2     17.1%          7.2%            49.9%            25.9%           
```

retiring是做有用功的比例，所有我们希望的这部分越高越好。bad speculation又分为
branch mispredicts和machine clears，前者是指错误的分支预测，后者是其它投机错误，
比如，提前指令的load操作，当遇到load/store冲突的时候，也要做flush，相当于这部分
投机执行时使用的硬件资源也白白浪费了。front-end bound分为fetch latency和fetch 
band-width, 这里不是很清楚。back-end bound分为core bound和memory bound，core 
bound是计算部件繁忙导致的bound，memory bound是存储繁忙导致的bound，memory bound
又分为store bound、各种cache bound以及ext memory bound等。

第7章
------

前端问题的调优方法(补充ARM上的具体例子，下同)。

第8章
------

后端问题的调优方法。注意，向量化的优化方法。
