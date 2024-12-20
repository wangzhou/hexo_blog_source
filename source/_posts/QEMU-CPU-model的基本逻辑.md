---
title: QEMU CPU model的基本逻辑
tags:
  - QEMU
description: >-
  本文分析QEMU中CPU model实现的具体逻辑，QEMU中主要是X86构架支持CPU model特性，
  所以文本的代码逻辑分析以X86代码代码为基础。分析基于QEMU v9.0.50版本。
abbrlink: 39617
date: 2024-08-05 23:36:59
categories:
---

基本逻辑
---------

虚拟机可以选择性的支持host上的功能，不同功能的集合可以表示不同的CPU类型，QEMU里
把用这样方式定义虚拟机CPU叫做CPU model。QEMU里X86定义了种类繁多的vCPU类型，具体
的说明可以参考[这里](https://qemu-project.gitlab.io/qemu/system/qemu-cpu-models.html)。

在基于KVM的QEMU上，需要KVM支持vCPU的特性可以被配置，具体逻辑可以参考[这里](https://wangzhou.github.io/Linux内核ARM64-cpufeature和errata的基本逻辑/)。

QEMU里需要定义不同类型vCPU的具体特性列表，并根据QEMU启动参数传入的CPU model类型
打开或者关闭对应的CPU特性。在基于KVM的QEMU上，打开或者关闭CPU特性是通过KVM的ioctl
KVM_SET_ONE_REG/KVM_GET_ONE_REG。

代码分析
---------

各种vCPU model定义在：static const X86CPUDefinition builtin_x86_defs[]。

如上的每个vCPU类型被依次注册到系统里：
```
/* target/i386/cpu.c */
x86_cpu_register_types
  +-> x86_register_cpudef_types(&builtin_x86_defs[i])
    +-> x86_register_cpu_model_type
      +-> type_register(&ti)
```
进一步展开如上的逻辑，可以看到每个vCPU类型被定义成一个class，注册到系统里。需要
注意的是，在QOM里，这里的type_register只是向系统注册一个class的描述，class本身的
创建还需要后续调用class_init回调函数。这里class的name就是定义的vCPU类型的名字，
这种类型的vCPU的父类就是TYPE_X86_CPU，这里类的继承关系是：
```
 CPUClass <--- X86CPUClass <--- 特定vCPU类
```
在特定vCPU类的初始化函数里填充X86CPUClass.model的数据结构，而model(即X86CPUModel)
是在x86_register_cpudef_types中创建的，其中包含有vCPU的具体定义信息。

X86 CPU的实例根据X86CPUClass中的定义被创建出来，在CPU实例初始化的时候把CPU的定义，
主要是支持feature的描述保存到CPU实例的env结构里(env结构描述整个CPU的状态)。
```
x86_cpu_initfn
 [...]
 +-> x86_cpu_load_model
   [...]
   +-> for (w = 0; w < FEATURE_WORDS; w++) {
           env->features[w] = def->features[w];
       }
```

基于KVM的QEMU上，KVM vCPU初始化的时候，依次根据env->features[]中的定义配置vCPU
的各个feature。(todo: 找到具体下配置的点?)
```
/* target/i386/kvm/kvm.c */
kvm_arch_init_vcpu
  +-> kvm_x86_build_cpuid
    +-> cpu_x86_cpuid
```
