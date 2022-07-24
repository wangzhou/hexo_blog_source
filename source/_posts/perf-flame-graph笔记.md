---
title: perf flame graph笔记
tags:
  - 软件调试
  - perf
description: 本文记录使用Linux下使用perf生成火焰图的方法
abbrlink: 864090c1
date: 2021-06-27 18:09:46
---

1. git clone https://github.com/brendangregg/FlameGraph

2. perf record -a -g -F 100
   这里-g是打点的时候函数调用都算，比如a函数调用了b，b里打的点也同时算在a里。
   -F是1秒中的打点次数，我们可以提高这个值来提高采样次数，比如这里的系统如果是
   16核，那么1s的采样数量就是16 × 100。

3. perf script > out.perf

4. cp out.perf /path_to/FlameGraph/

5. /path_to/FlameGraph/stackcollapse-perf.pl out.perf > out.folded

6. /path_to/FlameGraph/flamegraph.pl out.folded > out.svg

7. 然后可以在浏览器里打开out.svg
