---
title: qemu里增加trace的方法
tags:
  - QEMU
description: >-
  在调试qemu代码的时候可以在qemu的启动命令中增加--trace "xxx", 这样qemu代码
  运行到这个地方就会把相关的内容打印出来，这个文档介绍怎么新加一个这样的trace点。
abbrlink: b5567175
date: 2021-07-26 14:08:22
categories:
---

1. 在要加trace的模块对应的文件中加：#include "trace.h"

2. 在trace.h文件里写 #include "trace/trace-xxx_xxx.h"
    
   注意这里的连接符必须按照上面，其中的xxx是模块代码的路径，比如你的模块在qemu代码
   的hw/arm下，如上应该写成#include "trace/trace-hw_arm.h", 每一级路径都是下划线
   连接。

3. 在相同的目录创建trace-events文件，并在其中定义trace。定义的格式大概是这样的：
```
example(uint8_t level, uint32_t offset, uint64_t pte) "level: %u, pte offset: %u, pte value: 0x%lx"
```
   "example"是trace点的名字，括号里是各个参数的类型，引号里是输出的内容。

4. 在模块文件需要使用该trace点的地方使用 trace_example(xxx, xxx, xxx); 的方式调用。

详细的说明可以参考qemu的开发手册：https://qemu-project.gitlab.io/qemu/devel/tracing.html
