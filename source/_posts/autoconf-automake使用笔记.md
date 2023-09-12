---
title: autoconf/automake使用笔记
tags:
  - 软件开发
  - autoconf
description: 本文以一个简单的例子展示怎么使用autoconf/automake工具构建一个工程
abbrlink: fd4a67a2
date: 2021-07-17 11:22:30
categories:
---

1. 写hello.c程序

2. autoscan 生成：autoscan.log  configure.scan  hello.c
   cat configure.scan:
```
#                                               -*- Autoconf -*-
# Process this file with autoconf to produce a configure script.

AC_PREREQ([2.68])
AC_INIT([FULL-PACKAGE-NAME], [VERSION], [BUG-REPORT-ADDRESS])
AC_CONFIG_SRCDIR([hello.c])
        AC_CONFIG_HEADERS([config.h])

# Checks for programs.
AC_PROG_CC

# Checks for libraries.

# Checks for header files.

# Checks for typedefs, structures, and compiler characteristics.

# Checks for library functions.

AC_OUTPUT
```
修改成：
```
   #                                               -*- Autoconf -*-
   # Process this file with autoconf to produce a configure script.

   AC_PREREQ([2.68])
   AC_INIT([autoconf_test, [0.1]) # change this line!
   AC_CONFIG_SRCDIR([hello.c])
   AC_CONFIG_HEADERS([config.h])
   AM_INIT_AUTOMAKE([autoconf], [0.1]) # add this line!

   # Checks for programs.
   AC_PROG_CC

   # Checks for libraries.

   # Checks for header files.

   # Checks for typedefs, structures, and compiler characteristics.

   # Checks for library functions.

   AC_OUTPUT(Makefile) # change this line！
```
   把文件名改成: configure.in

3. aclocal 生成：
   aclocal.m4  autom4te.cache  configure.in  hello.c 
   (主要是生成aclocal.m4)

4. autoconf 生成：
   aclocal.m4  autom4te.cache  configure  configure.in  hello.c

5. autoheader 生成：
   aclocal.m4  autom4te.cache  config.h.in  configure  configure.in  hello.c

5. 创建Makefile.am:
   AUTOMAKE_OPTIONS=foreign
   bin_PROGRAMS=hello
   hello_SOURCES=hello.c

6. automake --add-missing 
   过程信息为：
   configure.in:8: installing `./install-sh'
   configure.in:8: installing `./missing'
   Makefile.am: installing `./depcomp'
   生成文件：
   aclocal.m4  autom4te.cache  config.h.in  configure  configure.in  depcomp  
   hello.c  install-sh  Makefile.am  Makefile.in  missing
   (makefile.in是这步生成的关键文件)

7. ./configure 生成最终的Makefile文件(该步骤中可能需要指定编译器：export CC=gcc)

8. make 生成最终的可执行的程序：hello, ./hello运行输出：test autoconf!
```
#include<stdio.h>

int main()
{
	printf("test autoconf!\n");
	
	#ifdef CONFIG_H_TEST
	printf("test autoconf: test config.h\n");
	#endif
	
	return 0;
}
```

注意，如上只是介绍了使用autoconf/autoheader/automake构建一个程序的流程，实际上
这三个工具是相互独立的，有各自独立完成的功能逻辑，我们也可以只使用其中的一个或者
几个工具来生成Makefile。
