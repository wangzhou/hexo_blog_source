---
title: SMMU SSV的逻辑
tags:
  - SMMU
  - 计算机体系结构
description: 本文描述SMMU协议中SSV的基本逻辑
abbrlink: 43e9d8f4
date: 2021-06-27 18:06:35
---

SMMU的STE表里的S1DSS=0b10时(目前的Linux主线内核代码是这样配置)，对于外设报文的
处理逻辑是：

 - a transction without a Substream ID is accepted and uses the CD of Substream 0.

 - a transction that arrive with Substream ID 0 are aborted and an event recorded.

上面SMMU判断一个外设上来的请求是否带有Substream ID的方法是看SMMU和外设之间的一个
叫SSV的信号有没有使能。这个SSV信号一般是外设可以配置的，如果一个外设把这个SSV
配置成1, 发给SMMU的请求中的PASID有是0的话, 就会报SMMU event错误。

常见硬件设计错误是把这个SSV的配置搞成设备全局的。这样这个设备只能要么工作在内核
态，要么工作在用户态，不能一部分资源工作在内核态，一部分工作在用户态。

正确的硬件设计是，把SSV的配置粒度减小。对于在内核态使用的资源，软件把对应的SSV配置
成0，对于在用户态使用的资源，软件把对应的SSV配置成1。
