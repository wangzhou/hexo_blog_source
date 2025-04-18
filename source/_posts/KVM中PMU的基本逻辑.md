---
title: KVM中PMU的基本逻辑
abbrlink: 13528
date: 2025-04-12 19:47:16
tags: [虚拟化, KVM, PMU]
description: "本文分析Linux内核KVM中ARM PMU模拟的基本逻辑。基于内核版本v6.11-rc7。"
categories:
---

基本逻辑
---------

KVM中对PMU的模拟思路和其他设备的模拟思路是一致的，通过device相关的ioctl创建设备
和配置设备属性，CPU在EL1访问PMU相关寄存器的时候会trap的KVM进行模拟，因为host/guest
是共享PMU物理硬件的，vCPU上下线时需要保存和恢复PMU寄存器的状态。

整体的ARM KVM框架逻辑可以参考[这里](https://wangzhou.github.io/Linux内核ARM64-KVM虚拟化基本逻辑/)。

注意一下几点，vPMU的初始化和配置通过KVM_ARM_VCPU_PMU_V3_*的ioctl进行，PMU相关的
系统寄存器的模拟逻辑在kvm_handle_sys_reg。

PMU vCPU上下线时保存和恢复寄存器的逻辑不是很直白，它应该在vCPU线程和perf相关的
sched_ini/sched_out的回调函数里。这个行为和KVM里PMU的模拟方式有直接的联系。

todo: PMU模拟方式。

PMU相关的寄存器总结如下：

  PMINTENSET_EL1
  PMINTENCLR_EL1
  
  PMCR_EL0              PMU控制寄存器
  
  PMCNTENSET_EL0
  PMCNTENCLR_EL0
  
  PMOVSCLR_EL0
  PMOVSSET_EL0
  
  PMSWINC_EL0
  PMSELR_EL0
  
  PMCEID0_EL0           event有无寄存器
  PMCEID1_EL0           event有无寄存器
  
  PMCCNTR_EL0
  
  PMXEVTYPER_EL0
  PMXEVCNTR_EL0

  PMEVTYPER<n>_EL0      配置对应counter的event
  PMEVCNTR<n>_EL0
  
  PMUSERENR_EL0
  PMCCNTR_EL0
  PMCCFILTR_EL0
  
  PMMIR_EL1

代码分析
---------

PMU相关的代码在arch/arm64/kvm/pmu.c、pmu-emul.c中。

vCPU创建，kvm_vm_ioctl_create_vcpu->kvm_arch_vcpu_create:
```
kvm_pmu_vcpu_init
```

vCPU init，kvm_arch_vcpu_ioctl_vcpu_init->kvm_vcpu_set_target->kvm_reset_vcpu:
```
kvm_pmu_vcpu_reset
```

对于vCPU fd ioctl的KVM_HAS/SET/GET_DEVICE_ATTR:
```
KVM_SET_DEVICE_ATTR/KVM_ARM_VCPU_PMU_V3_CTRL -> kvm_arm_pmu_v3_set_attr:

KVM_ARM_VCPU_PMU_V3_IRQ
KVM_ARM_VCPU_PMU_V3_FILTER
KVM_ARM_VCPU_PMU_V3_SET_PMU
KVM_ARM_VCPU_PMU_V3_INIT

KVM_GET_DEVICE_ATTR/KVM_ARM_VCPU_PMU_V3_CTRL -> kvm_arm_pmu_v3_get_attr:

KVM_ARM_VCPU_PMU_V3_IRQ
```

PMU的模拟逻辑。

杂项问题
---------

KVM对guest呈现的PMU version的问题。KVM从host获得PMU的硬件版本，并把这个信息做一定
的限制后保存在kvm->kvm_arch->id_reg.dfr0寄存器中，后续KVM都是从这里拿vPMU的版本
信息。
```
/* vCPU初始化的时候，在reset里从host拿信息并保存到如上KVM结构里 */
read_sanitised_id_aa64dfr0_el1
  +-> val |= SYS_FIELD_PREP(ID_AA64DFR0_EL1, PMUVer, kvm_arm_pmu_get_pmuver_limit());
```


