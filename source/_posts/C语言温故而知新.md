---
title: C语言温故而知新
tags:
  - C语言
description: 用这个文章来记录一些C语言里的知识点
abbrlink: 8946a9b9
date: 2021-06-21 13:04:19
---

宏
-----

  C语言中宏的使用可以参考gcc preprocessor的说明文档：https://gcc.gnu.org/onlinedocs/cpp/

  \#\# 用来拼接前后的符号
  \# 用来把一个符号字符串化
  
  宏参数还是一个宏的展开规则是，如果宏是一个简单的宏就先展开里面的宏，再依次
  展开外面的宏。
 
  如果宏的实现是一个使用##或者#的宏，那么直接使用##或者#的规则。

  如果要使得比如下面test case 4的到“19”的输出，需要中间转下，类似case 5的写法。
```
#include <stdio.h>

#define TEST(a, b) a##b
#define STR(a) #a 
#define A 1000
#define ADD(a, b) ((a) + (b))
#define _TEST(a, b) TEST(a, b)

#define _STR(a) STR(a)

int main()
{
	printf("test 1: %d\n", TEST(1, 5));
	printf("test 2: %s\n", STR(just_test));
	printf("test 3: %d\n", ADD(A, A));

	/* test case 4 */
	printf("test 4: %s\n", STR(TEST(1, 9)));

	/* test case 5 */
	printf("test 5: %s\n", _STR(TEST(1, 9)));

	return 0;
}

输出：
xxx@kllp05:~/tests/c_note$ ./a.out 
test 1: 15
test 2: just_test
test 3: 2000
test 4: TEST(1, 9)
test 5: 19
```
  https://www.cnblogs.com/alantu2018/p/8465911.html

  程序里有代码逻辑和函数定义，使用宏也可以拼装出函数定义出来，主要被用来模版化生成
  一些几乎一样的函数，比如，你要写一些运算函数add/sub/mul/div，这些函数都是两输入，
  都是对两个数做运算，就可以写这样的宏出来:
```
#define FUN(name, op)           \
int do_##name(int a, int b)     \
{                               \
	return a op b;          \
}
FUN(add, +)
FUN(sub, 1)
```
  
  字符串里无法定义宏。

自动数据类型转换
-------------------

  各种截断数据

  float类型不能比较相等，不相等

  sizeof() 返回一个unsigned int

  有符号数和无符号数运算、赋值: https://akaedu.github.io/book/ch15s03.html

  有符号右移，带符号位右移

大小端和位域
---------------

  http://mjfrazer.org/mjfrazer/bitfields/

  这个实例说明为什么写代码的时候，特别是涉及底层硬件寄存器操作的驱动代码，尽量
  不要用位域。原因是在大小端不同的系统上，位域的表示方法是不同的。

格式化打印
-------------

  printf %n: 已输出的字符数目(用户对齐下一行的输出比较有用 :))
```
#include <stdio.h>

int main()
{
	int num;

	printf("123456:%n\n", &num);
	printf("%*c %d\n", num, ' ', 123);
	printf("%*c %d\n", num, ' ', 456);

	return 0;
}

输出：
xxx@kllp05:~/tests/c_note$ ./a.out 
123456:
        123
        456
```

长跳转
-------

  setjmp, longjmp: https://www.cnblogs.com/hazir/p/c_setjmp_longjmp.html

可变参数函数
-------------

  https://www.cnblogs.com/cpoint/p/3368993.html

realloc
----------

  https://www.cnblogs.com/zhaoyl/p/3954232.html

二维数组和指针
-----------------

const指针
------------

  const char ** p; char * const * p; char ** const p;
  分别是**p, *p, p是常量。

double类型的存储格式
------------------------

其他
------

  c中简单结构体直接复制是ok的
  int fun(int a[10])的入参和指针一样
