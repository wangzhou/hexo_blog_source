---
title: 搭建邮件客户端进行Linux kernel开发
tags:
  - 软件开发
description: 本文介绍用纯命令行处理邮件的环境搭建方法
abbrlink: 6819ae4f
date: 2021-07-11 23:45:12
categories:
---

进行linux kernel开发需要实时关注kernel相关邮件列表的动态，本文介绍怎么搭建一个
邮件客户端，以便更好的查看邮件列表中的邮件。根据本文内容可以搭建一个这样的邮件
阅读环境。

收到内核邮件列表中的邮件需要首先订阅相应的邮件列表，具体订阅的方法请自行google.
订阅的本质是向一个地址发送特定内容的邮件。

每天来自一个邮件列表中的邮件可能会有几十封，如果同时订阅几个邮件列表，每天邮箱
中的邮件会有几百封。怎么管理这些邮件，有一种方法是用雷鸟邮件客户端(thunderbird),
还有一种方法就是本文将要介绍的 fetchmail + procmail + msmtp + mutt。当然你也可以
直接从你注册的邮箱里看邮件，不过从成千上百的邮件中找到你想看的，再找到别人的回复
的邮寄真的非常麻烦。

除了邮件客户端，还有一个问题：用什么邮箱。一般的公司邮箱可以用来注册邮件列表，
但是业余时间也想学习下大牛们的讨论，用公司邮箱显然不太好。于是需要注册一个私人
邮箱，用gmail显然要显得专业，但是用gmail邮箱设置thunderbird的时候比较麻烦，要先
生成一个第三方app接入gmail邮箱的密码，在设置thunderbird密码的时候设置进去，本人
用 gmail + thunderbird 的方式搭的环境，收邮件的时候时而连上时而连不上，所以本文
对这个不做介绍。用163,126邮箱注册了邮件列表，发了邮件就没有了下文，估计是给屏蔽
了，大家也不用忙活用163，126邮箱注册邮件列表了。在网上发现有人用139的邮箱成功
注册内核邮件列表，试了一下，果然可以。所以本文使用139邮箱。(update: 139用了一段
时间也不行了)

有了上面的背景，我们可以开始介绍整个环境的搭建了。首先想到的肯定是 139邮箱 + thunderbird
的方法，thunderbird图形化操作似乎也更方便一点。thunderbird只是一个邮件客户端，你
要配置它，告诉它发送/接收邮件使用的发送和接收服务器分别是什么。139邮箱的发送接收
服务器可以在邮箱的设置里找到，至于怎么设置thunderbird, 具体方法请自行google.
用thunderbird的好处是可以设置不同的邮件目录，对应的目录设置不同的过滤器以归类
邮件，还有就是可以以一个个thread的形式管理邮件，回复关系一目了然。需要注意的是
thunderbird的默认格式不符合linux kernel patch的格式，需要简单配置一下，想用
thunderbird的人也要google下了。

但是像本人这样，有时需要远程调试，然后在服务器上把调试log发回本地的情况时。图形
界面的邮件客户端显然满足不了需求，于是有了本文要介绍的命令行邮件客户端配置。
本文使用139邮箱，139邮箱默认是开启pop, smtp服务的，下面的配置脚本中可以看到
pop, smtp服务器地址。

首先简要说明fetchmail, procmail, msmtp, mutt的作用。
fetchmail是从邮箱下在邮件, procmail提供对邮件的过滤功能, msmtp用来发送邮件，
mutt用来看邮件和写邮件。本人的工作环境是ubuntu, 所以下面配置都是在ubuntu的环境
下完成的。ubuntu下安装上面四个软件apt-get install即可搞定。

fetchmail的配置文件在自己的home目录下的.fetchmailrc文件，需要自己创建一个这样
的文件。文件内容大概如下：

```
poll pop.139.com
protocol POP3
user "your_email_address@139.com"
password "your_password"
keep
ssl
mda "procmail -d %T"
```

配置完后，运行fetchmail, 可以看到在/var/mail/下有以你账号名作为文件名的文件。
里面就是下载的邮件。本人一开始没有配置最后一行，下载邮件失败, 加上最后一行后可以
下载。

procmail的配置文件也在自己的home目录下，为.procmailrc， 内容大概如下：
```
PATH=/usr/local/bin:/usr/bin:/bin
MAILDIR=$HOME/Mail
DEFAULT=$MAILDIR/mbox
LOGFILE=$MAILDIR/log
```

msmtp的配置文件也在自己的home目录下，为.msmtprc， 内容大概如下：
```
account default
host smtp.139.com
from your_email_address@139.com
auth login
tls on
tls_certcheck off
user your_email_address@139.com
password "your_password"
```

配置好后可以用 msmtp -S 检查自己的配置是否正确。

mutt的配置文件也在自己的home目录下，为.muttrc， 内容大概如下：
```
set from="your name <your_email_address@139.com>"
set use_from=yes
set sendmail="/usr/bin/msmtp"
set editor=vim
set folder="$HOME/Mail"
set record="+sent"
set mbox="+mbox"
set postponed="+postponed"
```

配置好后每次可以先fetchmail, 然后用mutt -f ~/Mail/mbox打开邮件查看。

到现在就基本可以命令行浏览，收发邮件了。但是如果你一下订阅了几个邮件列表，想
分类管理邮件，下面给一个简单的示范：

想单独管理下来自linux-pci@vger.kernel.org 邮件列表里的邮件。先在家目录的Mail下
创建一个文件保存邮件，这里我们创建一个叫pci-mbox的文件。在.procmailrc文件中添加
一下几行：
```
:0
* ^Cc:.*linux-pci@vger.kernel.org
pci-mbox
```
之后再fetchmail，来自pci邮件列表中的邮件就自动保存在~/Mail/pci-mbox中了。

注意：
1. fetchmail失败: 在.procmailrc中配置不当（配置不当的过滤脚本）会导致fetchmail
   下载不了邮件
