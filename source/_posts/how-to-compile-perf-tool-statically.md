---
title: how to compile perf tool statically
tags:
  - perf
description: 一个静态编译perf的流水帐
abbrlink: 235c920b
date: 2021-07-05 22:28:32
categories:
---

1. D05 machine + CentOS7.4

2. kernel code

3. cd kernel/tools
   make LDFLAGS=-static perf

4. you can find perf statically in kernel/tools/perf

Note:

1. you may need to: yum install glibc-static
2. you need install slang to enable perf report -tui. In ubuntu20.04, you need
   sudo apt install libslang2-dev. you can use ldd perf to see if perf already
   links slang
3. you may need to install a lot other libs to compile a full function perf.
   (libelf-dev)
