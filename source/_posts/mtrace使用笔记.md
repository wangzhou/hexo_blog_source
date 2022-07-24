---
title: mtrace使用笔记
tags:
  - 软件调试
description: >-
  mtrace是一个GNU C库中的内存检测工具，用以检测用户态程序内存泄露 (malloc, realloc,
  free)。检测的原理为在程序开始时通过mtrace()为malloc等函数安装 handlers[1],
  malloc等函数执行时就会把相应的信息写到指定文件中(由环境变量指定)[2]， 最后由一个perl脚本(/usr/bin/mtrace)解析信息。
abbrlink: d5ac76ae
date: 2021-07-17 11:24:05
categories:
---

# 使用过程：

1. 程序中需要包含头文件mchech.h, 在程序开始处调用mtrace()
2. 设定环境变量 export MALLOC_TRACE="mtrace.out"
3. 编译运行程序, 会生成mtrace.out文件
4. mtrace a.out mtrace.out得到内存泄露信息

Memory not freed:
-----------------
Address     Size     Caller
0x0000000001650490     0x28  at /vm/***/src/mtrace_test/mtrace_test.c:11
0x00000000016504f0     0x28  at /vm/***/src/mtrace_test/mtrace_test.c:13
0x0000000001650550      0xa  at /vm/***/src/mtrace_test/mtrace_test.c:15
```
/* mtrace_test.c */
#include <stdio.h>
#include <stdlib.h>
#include <mcheck.h>
#
int main(void)
{
	mtrace(); 
	
	int i;
	int *p_0 = (int*)malloc(sizeof(int)*10);
	int *p_1 = (int*)malloc(sizeof(int)*10);
	int *p_2 = (int*)malloc(sizeof(int)*10);
	int *p_3 = (int*)malloc(sizeof(int)*10);
	int *p_4 = (int*)malloc(sizeof(int)*10);
	char *p_char = (char*)malloc(sizeof(char)*10);
	
	for (i = 0; i < 10; i++) {
		p_0[i] = i;
	}

	for (i = 0; i < 10; i++) {
		p_char[i] = 'w';
	}

	free(p_0);
	/* 制造人为的内存泄漏 */
	//free(p_1)
	free(p_2);
	//free(p_3)
	free(p_4);
	//free(p_char);
	
	return 0;
}
```

# 参考：
[1] http://www.gnu.org/software/libc/manual/html_node/Allocation-Debugging.html
[2] http://www.gnu.org/software/libc/manual/html_node/Hooks-for-Malloc.html
