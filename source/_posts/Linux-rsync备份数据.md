---
title: Linux rsync备份数据
tags:
  - 运维
  - shell
description: '为了实现数据的实时备份，在linux可以使用rsync, 本文是使用rsync时的一个笔记 这里使用的系统是ubuntu 14.04'
abbrlink: cfcf7e3f
date: 2021-07-17 10:55:00
categories:
---

 需要备份的数据在服务器上，这里叫做server。把数据备份到client上，这里也叫
 backup server。下面是具体的步骤：

步骤：
0. run "sudo apt-get install rsync", if do not have one
1. configure /etc/rsyncd.conf in server, need to touch a file yourself.
2. configure /home/***/security/rsync.pass in server indicating client's
   user_name and key.
3. configure /etc/default/rsync: RSYNC_ENABLE=true in server
4. run "sudo /etc/init.d/rsync start" in server
5. run "rsync -vzrtopg --progress user_name@server_ip::test /home/test" in client
   (backup server) to backup directory indicating in [test] in /etc/rsyncd.conf
   in server to /home/test in client, using user user_name

说明：
1. 遇到这样的错误"auth failed on module xxx", 可以查看：
   http://blog.sina.com.cn/s/blog_4da051a60101h8am.html
2. 配置中遇到错误，可以在/var/log/rsyncd.log查看log记录
3. /etc/rsyncd.conf中的配置([test]), 可以参考：
   http://www.iteye.com/topic/604436
4. 在ubuntu 14.04上，/etc/rsyncd.conf中的pid file为：/var/run/rsyncd.pid
5. 步骤1～4在服务器上完成，其中user_name是client执行备份命令时的用户名，key是密码
6. 步骤5说明在client上应该用什么样的命令备份服务器上的目录，user_name是步骤2中
   的user_name, server_ip是服务器的ip, test是在/etc/rsyncd.conf中的[test],
   /home/test表示把[test]中指示的目录备份到client的/home/test下
7. 具体可以根据需求把步骤5中的命令在每天的某个时候定时执行，这样每天的数据都得到了备份
