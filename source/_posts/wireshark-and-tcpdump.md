---
title: wireshark and tcpdump
tags:
  - 网络
  - wireshark
description: 本文记录使用wireshark做tcpdump的简单步骤
abbrlink: b5ef2a9
date: 2021-07-11 23:32:47
categories:
---

tcpdump可以捕获某个网络端口的所有的网络报文。可以用：
```
tcpdump -i eth0 -w dump_file
```
捕获从eth0经过的网络报文，然后把他们的具体信息存在dump_file指定的文件中。

用wireshark可以打开dump_file文件解析其中的报文。wireshark是一个图形化的网络报文
分析工具。在windows和linux下都有wireshark的版本。

一般的，看报文的统计结果的时候，比如看有多少重传的TCP报文，可以在wireshark中
打开Analyze -> Export Info -> Notes 查看。
