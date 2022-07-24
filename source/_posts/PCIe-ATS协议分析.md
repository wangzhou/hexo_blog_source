---
title: PCIe ATS协议分析
tags:
  - PCIe
  - 计算机体系结构
description: >-
  本文具体分析ATS相关的一些消息格式, 以及一些相关的设计逻辑。这些消息格式
  除了capability，其他的都是软件不感知的，但是通过分析他们，我们对系统的 行为可以有更深一点的了解。当前的版本中，我们先分析目前比较关心的域段，随后
  看需要补充其他的域段。
abbrlink: f0d7cd34
date: 2021-09-17 23:09:55
categories:
---

translation request
-------------------

 - Length

   这个地方的数值表示的是dword的个数，因为一个tranlation request是两个dword，所以
   如果Length是2，就表示请求untranslated address开始的一个STU大小的翻译，如果，
   大于2就表示Length/2个STU大小的翻译。可见Length总是偶数。

   Spec里说，这样申请的多个翻译需要是一样的页大小，以STU为大小，从untranslated
   address起始。

   Length的最大取值是RCB，就是128，那么一次申请的翻译最大只有64个SUT。

 - Untranslated address

   需要翻译的VA，size大小是 (2^(STU + 12)) * (Length / 2) Byte。STU就是ATS
   capability里的STU域段的数值。这个VA总是要和STU表示的大小对齐。

translation completion
----------------------

 翻译的结果用这个消息返回给PCIe设备。如果翻译成功，在completion的报文中就带有
 payload，payload中有Translated address，S，N，Global，W，R等域段。

 - Status
  
   表示翻译的结果。

 - Translated address和S
   
   翻译后的PA，这个地址覆盖的size需要S域段和地址本身确定。S为0表示size的值是4096，
   S为1就要看PA其中的bit了，bit12为0表示8KB，bit13/bit12为0/1表示16KB，依次类推
   Spec上定义了2M，1G，4G的表示方法。

 - Global

   表示翻译得到的结果的作用范围，如果是1，表示是整个设备，如果是0，设备需要配合
   PASID给这个翻译项创建ATC，这样范围就被限定在了PASID的范围内。

 按照如上的逻辑，translation request的size只能和STU的大小一样，而STU是ATS capability
 里在设备初始化时配置好的，所以size的大小在设备的使用过程中是固定的。

invalidate request
------------------

 无效化一段VA在ATC的缓存。

 - Untranslated address和S
 
   地址和S域段和上面的意思一样。不同的是无效化的size只能是2的幂次方倍个4096。
   Spec中没有明确说Length域段怎么和其对应起来。

   SMMUv3 Spec的CMD_ATC_INV命令，使用4096 * 2^size来指定无效化的大小，和PCIe这里
   是配套的。这样设定无效化的size，一般是和实际需要无效化的size是不一致的，需要
   软件去匹配一下。

 - Global

   和上面的意思一样，表示PASID这个因素要不要加到invalidate里来。

ATS capability
--------------

 - cap head

   常规head

 - Capability register

   * Invalidate Queue Depth，不清楚暴露给软件的目的？

 - Control register

   * STU
