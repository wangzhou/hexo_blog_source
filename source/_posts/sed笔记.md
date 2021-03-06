---
title: sed笔记
tags:
  - shell
description: 本文是Linux sed命令的一个学习笔记
abbrlink: 1875a13a
date: 2021-07-11 23:33:46
categories:
---

sed是一个面向数据流的编辑器。和vim这种交互式编辑器不用，sed按照定义的处理模式处理
每一行文本。比如：
```
/* 把test_file文件中每行的第一个'haha'换成'taoge' */
sed 's/haha/taoge/' test_file
```
其中s/haha/taoge就是定义的处理模式，test_files是要处理的文件。sed把处理后的文本
向标准输出，不会更改原文件。

一个sed命令还可以在sed和处理模式之间加入命令选项。比如：
```
sed -f sed_script test_file
```
表示用sed_script中的处理模式（每行一个处理模式）处理test_file中的所有行。
又比如：
```
sed -e 's/this/a/; s/that/b/' test
```
-e后面是分号隔开的多个匹配模式。

sed编辑器的处理模式中常用的命令有s, d, i, a, c, w, p, r, 寻址...

s命令比较常见，即像上面展示的：s/old/new/. vim的修改命令也是这样的，一般的，
linux邮件列表里指示修改的评论有时候也写成sed命令的样子了。都是过来人，行业"黑话"
一看都懂。s命令的最后面还可以跟一些flag, 比如s/old/new/g, 表示整行所有的old都替换
成new; s/old/new/2, 表示替换第二个old成new; s/old/new/p, 表示输出发生替换的行，
一般常常是：sed -n s/old/new/p，-n表示结果不输出，于是最终结果是只输出了发生替换
的行; s/old/new/w file, 表示把输出的结果保存在file文件中。

想指定具体行的修改，可以寻址方式修改, 比如：2s/old/new/, 2,3s/old/new, 1,$s/old/new
分别是修改第二行，第二行和第三行，第一行到最后一行。vim里格式是一样的。

更加一般的，这里的寻址方式也可以换成匹配字符, 比如：/wang/s/old/new/
找见有字符串'wang'的一行, 在这一行用s命令把第一个old改成new. 这个匹配特定行，再
处理的方式也可以在除了s命令的其他命令上使用。

d是删除命令。上面熟悉的格式可以套用到d命令上：sed '1d' test, sed '1,3d' test,
sed '/wang/d' test, sed -e "/wang/d; 1d" test, 分别是删除第一行，删除1~3行，删除
含有'wang'的一行，删除含有'wang'的一行和第一行。

i, a, c是插入, 附加操作和更改操作, 比如：sed '2i new_line' test,
sed '2a new_line' test, sed '2a new_line' test. 以上各自是在test文件的第二行之前
插入new_line这行，在test文件第二行之后插入new_line这行，直接把test文件的第二行改成
new_line这行。

y是变换命令。可以完成类似文本加密的工作, 当然这里说的加密只是一对一的做字符替换,
比如：sed 'y/123/579/' test, 可以把test中所有的字符1,2,3依次替换成5,7,9

之前提到flag p, p用于输出指定的值，一般都结合-n使用，-n用于禁止输出，这样就可以
控制输出想要输出的东西了，比如：sed -n  '/wang/p' test, sed -n '2,3p' test
输出有'wang'字符的行，输出第2行和第三行。

之前也提到了flag w, w用于把改动写入到文件, 比如：sed '1,3w new_file' test,
sed '/wang/w new_file' test, 把test文件的1~3行写入到new_file中, 把test文件中
含有'wang'的一行写入到new_file文件中。

和w相对的还有读命令，比如： sed '3r new' file, 读出new文件中的内容然后加到file
文件的第三行之后。当然也可以 sed '/wang/r new' file, 读出new文件中的内容，加到file
文件含有'wang'这一行之后。
