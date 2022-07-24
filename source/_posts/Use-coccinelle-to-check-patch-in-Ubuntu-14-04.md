---
title: Use coccinelle to check patch in Ubuntu 14.04
tags:
  - 软件开发
description: >-
  In Ubuntu14.04, when running coccinelle, it will fail. So we install it
  manually to make patch check. This note helps to remember the steps.
abbrlink: d2b5a282
date: 2021-07-05 22:42:02
categories:
---

1. install from its git repo: https://github.com/coccinelle/coccinelle.git

2. install the packedge mention in install.txt

3. ./autogen
   ./configure
   ./make
   ./make install

4. cd in a kernel directory, run:
 
   make coccicheck MODE=report M=arch/arm64/mm

   It will check all the patch under, e.g. arch/arm64/mm
