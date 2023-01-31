---
title: ftp服务器设置
tags:
  - 运维
  - shell
description: 本文记录一次ftp服务器的搭建过程，同时提供一种Linux上查找资料的方法
abbrlink: 738a10a8
date: 2021-07-17 11:17:36
categories:
---

做嵌入式调试需要在主机上建立一个ftp服务器，然后通过网线将主机上的程序下载到嵌入
式开发板上，主机的ftp服务器需要有用户名和密码。以上是所知道的信息，怎么利用现有
的资源快速解决这个简单的问题？上网搜索是必要的，但是漫无目的的搜索效率并不是很
大；还有一种方法是系统的把ftp服务器的知识学一下，这样有太慢。这里提供几个工具和
思路，或许会比较有用。

1. man -k ftp，man命令加-k选项可以列出所有ftp相关的帮助信息，我们可以从中选择
  运行这个命令后可以看到ftp，tftp，vsftpd等相关项目。可以以上面的信息作为基础
  再在网上搜索

2. 可以看到vsftpd是一个FTP服务器，man 8 vsftpd可以查看和他相关的信息

3. 在google中搜vsftpd的信息，输入ubuntu vsftpd，第一条就得到下面的信息：
   https://help.ubuntu.com/10.04/serverguide/ftp-server.html

   文件的头几行就得到这样的信息：
```
   Access to an FTP server can be managed in two ways:
   Anonymous
   Authenticated
```
   上面告诉我们FTP服务器分为两大类：匿名的(就是直接ftp \<ip\>就可以登陆的)，需要
   输入用户名、密码的。结合我们的需要，我们的搜索词变成了ftp，authenticated，但是
   这篇文档已经介绍了需要怎么设置，我们就不需要去别的地方搜了。下面的"User Authenticated
   FTP Configuration"说明要设置/etc/vsftpd.conf中的：
```
          local_enable=YES
          write_enable=YES
```
   然后在重启vsftpd:
```
          sudo /etc/init.d/vsftpd restart
```

4. 说到这里我们的pc(ubuntu系统)上还没有vsftpd啊，试试ubuntu的软件下载管理工具
   sudo apt-get install vsftpd(可以自动补全)，果然可以。

5. vsftpd服务器有了，开始配置/etc/vsftpd.conf。我们打开对应的文件
   sudo vi /etc/vsftpd.conf
   仔细看，发现文档已经是充满注释了。找个和我们目的相关的：
```
   # Allow anonymous FTP? (Beware - allowed by default if you comment this out).
   anonymous_enable=NO
```
   后面的注释说，这个是默认带开的，我们用的是authenticate，所以这里选NO

6. 按这样的设置，然后重启服务器，用ftp 127.0.0.1登陆自己的服务器，发现要输入的
   自己ubuntu系统的用户名和密码作为vsftpd的用户名和密码，但是到了哪个目录中了呢？
   用get <自己home中的文件>，发现可以把自己home中的文件拉到当前目录下，看来默认
   vsftpd的目录就是自己的/home/XXX

7. 既然/etc/vsftpd.conf文件注释很好，那就到该文件中看看怎么设置，设置：
   local_root=/home/XXX/your_ftpboot
   chmod 777 /home/XXX/your_ftpboot
   重启服务器，发现vsftpd可以使用了，根目录就是上面设置的

   至于想更好的用好ftp服务器，就是研究、尝试/etc/vsftpd.conf的事了
