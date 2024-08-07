---
title: 尝试使用gem5/konata跟踪CPU流水线
tags:
  - gem5
  - Konata
  - 计算机体系结构
description: 本文记录使用gem5输出riscv处理器pipeline trace的一个尝试。
abbrlink: 45601
date: 2023-06-18 17:27:49
categories:
---

背景介绍
---------

gem5是业界常用的CPU性能仿真模型，和qemu不同，gem5可以模拟CPU内部的微架构。gem5模拟
过程可以记录各种日志信息，帮助CPU开发人员了解CPU的实际行状况，我们这里介绍的是gem5
运行过程记录CPU流水线各个阶段的时间信息。使用图形化工具可以把记录的信息在图上画出
来，人们可以非常直观的看到CPU的整个运行过程，这里我们使用的是一个叫Konata的画图工具。

具体步骤
---------

整个步骤大概分为：1. 下载并编译gem5; 2. 下载并配置Konata；3. 用一个简单程序进行
展示。

可以从官网下载gem5，不过我是从<https://gitee.com/koverlu/gem5.git>下载的，可能会快
一点。测试是在ARM64 ubuntu 20.04下进行的，一口气apt install了全部gem5依赖的库。
然后在gem5代码根目录下使用：scons build/RISCV/gem5.opt -j4 编译riscv处理器的gem5。
我的编译机器就是做测试的机器，是一台M1的Macbook Air上的虚拟机，系统是ubuntu 20.04，
整个编译可能要十几分钟，需要注意的是，gem5链接需要的内存比较大，我的虚拟机是4GB
内存，链接会报错，调整到8GB就可以完成链接了。

从<https://github.com/shioyadan/Konata.git>下载Konata。直接运行Konata代码根目录下的
install.sh，然后在ubuntu的图形化界面下打开terminal，并在其中运行Konata代码下的konata.sh，
这时就会跳出Konata的图形化界面，可以通过File->open打开Konata自带的log文件，log文件
的路径在Konata/docs/kanata-sample-[12].png，需要解压缩一下得到log文件，一开始是压缩的。

随意写一个简单代码，然后静态编译下：riscv64-linux-gnu-gcc --static test.c -o test
```
int add(int a, int b)
{
	return a + b;
}

int main()
{
	int a = 1, b = 2, c = 0;
	
	c = add(1, 2);

	return 0;
}
```
在gem5代码根目录下运行：
```
./build/RISCV/gem5.opt --debug-flags=O3PipeView --debug-file=trace.out configs/example/se.py --cpu-type=O3CPU --caches --cmd /home/sherlock/test
```
输出log在gem5/m5out/trace.out，用如上同样的方法在Konata里打开trace.out，就可以得到
如下的pipeline图形化输出:
![Konata展示gem5 pipeline](gem5_konata.png)
上图中横坐标是时间，单位是CPU的cycle，纵坐标是指令进入流水线的先后顺序，单位就是
一条指令，横坐标每个不同的颜色表示不同的流水线阶段，比如F是fetch、Dc是decode、Rn
是Rename、Is是issue、Cm是commit。图中明亮的部分是流水线中正常退休的指令，灰色部分
是被flush掉的指令，比如一开始CPU一拍取8条指令，实际上第一条指令就是一个跳转指令，
后面的1-15条指令都取错了，后面CPU检测到分支预测失败后就会flush掉之前错误取入的指令。

Konata调试
-----------

如果你自己hack了Konata这个软件，怎么去调试它呢？在它的help有Toggle Dev Tool这个按钮，
点击它就可以显示出调试界面，调试界面点击source可以打开Konata的代码，你可以点击代码
行数位置添加断点，如果简单的问题可能可以大概猜到出问题的地方，遇到复杂的问题就要
去熟悉JS前端的知识了。
