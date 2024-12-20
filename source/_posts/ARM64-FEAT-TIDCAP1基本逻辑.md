---
title: ARM64 FEAT_TIDCAP1基本逻辑
tags:
  - ARM64
description: 本文是ARM64 FEAT_TIDCAP1基本逻辑整理的一个笔记。
abbrlink: 10001
date: 2024-01-21 21:34:14
categories:
---

这个特性是ARMv8.8的一个mandotary的特性，基本逻辑是开启该特性的时候，ARM为其他厂商
预留的指令空间(Reserved encoding space for IMPLEMENTATION DEFINED instructions)
上如果定义了指令，CPU在执行到相关自定义指令时就会报异常。

在Linux主线内核上(5.19)合入了相关patch(arm64: trap implementation defined functionality
in userspace)，这个patch检测FEAT_TIDCAP1，如果有就直接打开这个特性。

这个特性相当于在Linux主线上断绝掉了其他厂商在ARM系统架构里自定指令的可能。
