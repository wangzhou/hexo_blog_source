---
title: 使用hexo NexT搭建个人博客
tags:
  - 静态博客搭建
description: >-
  这篇文章记录一个简单个人博客的搭建过程，我们使用GitHub page + hexo + NexT主题的搭建方式。 整个搭建的环境是在一台arm64
  ubuntu20.04机器上。
abbrlink: 73f1d6bc
date: 2021-06-18 23:36:31
---

1. 基础逻辑说明
---------------

 github提供了静态网页展示的功能，我们可以在自己的github上创建username.github.io
 名字的仓库，然后在这个仓库的settings里开启Pages的功能。这里需要html文件才能以
 静态网页的形式把内容展示出来，而一般我们直接书写的是md或者是rst这样的文本文件。
 
 网上有各种各样的从文本文件生成html文件的框架，hexo就是其中的一个，它是基于JS的。
 hexo支持主题可选，通过选不同的主题，博客的风格会不一样，我们选的是NexT主题，这个
 主题使用的人很多，相关的资料也容易找。

 我们把整个blog分成三个代码仓库来管理：

 * hexo代码仓库，这个仓库是hexo代码，我们的博客原文也放在这个目录下。

 * 静态网页仓库，这个仓库里的静态网页都是hexo生成的。我们把这个仓库推到如上的
   username.github.io这个仓库，然后从浏览器就可有通过https://username.github.io
   访问到。

 * NexT主题的仓库，我们把这个仓库fork一份到自己的github上，一般我们只需要改动
   NexT的配置文件就好，但是我们这里还是把仓库本身也保存一份，日后可以直接修改NexT
   的代码，然后提交到自己的仓库暂时保存。我们把自己的这个NexT仓库作为hexo仓库的
   submodule保存。

2. 搭建过程
-----------

 * 安装npm、nodejs和hexo

   sudo apt install npm nodejs
   sudo npm install -g hexo-cli

   Note: 上面安装的nodejs是10.x.x的，后面生成博客的时候会报错，我们这里需要安装
         12.x.x的nodejs版本。nodejs的版本可以通过node -v 来看。

 * 安装12.x.x nodejs

   curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
   sudp apt update
   sudo apt install -y nodejs

 * 下载hexo和NexT

   mkdir hexo; cd hexo
   hexo init blog
   cd blog
   sudo npm install

   把NexT fork到自己的github, 我们用url_priv_next表示自己NexT的github仓库地址。
   创建一个空的github仓库用来存放hexo代码, 我们用url_priv_hexo表示自己的这个hexo
   仓库的github地址。在hexo仓库把自己的NexT仓库设置成自己的hexo仓库的submodule:

   git submodule add url_priv_next themes/next

   到这里为止，基本的仓库以及他们之间的关系我们都搞定了。我们看看hexo里几个目录
   里放什么：source里放博客相关的原文件，其中包括md文件和相关的图片；themes放各种
   主题，所以NexT也放到这个目录下；scaffolds放各种模版，比如后面会讲到的生成文章
   就会默认用到scaffolds/post.md这个模版，所以我们更改这个模版生成的文章也会带上
   相关的改动，如下的description就是这样的一个例子, draft.md这个模版可以用来生成
   草稿文档，比如我们可以用hexo new draft "draft_1"来生成名字是draft_1的草稿文档，
   这个草稿文档以draft.md为模版，存放在source/_drafts下；public用来放后面生成
   的html文档；node_modules是一些和Hexo nodejs相关的东西。

 * 配置hexo以及NexT的配置文件

   如上的配置已经可以生成静态网页。出于个人偏好，我自己的配置文件的改动如下，
   具体的改动可以看相关地方的注释：
```
hexo _config.yml

diff --git a/_config.yml b/_config.yml
index 2dc35e6..97fc902 100644
--- a/_config.yml
+++ b/_config.yml
@@ -97,7 +97,7 @@ ignore:
 # Extensions
 ## Plugins: https://hexo.io/plugins/
 ## Themes: https://hexo.io/themes/
-theme: landscape
+theme: next  # 使用NexT主题

diff --git a/scaffolds/post.md b/scaffolds/post.md
index 1f9b9a4..91f86d0 100644
--- a/scaffolds/post.md
+++ b/scaffolds/post.md
@@ -2,4 +2,5 @@
 title: {{ title }}
 date: {{ date }}
 tags:
+description:    # 如下使用hexo g 生成新md文档的时候会使用这个地方的模版，如果这里加上description
                 # 主页就不会展示全部文件，而是只是显示这里的摘要。

```
```
NexT _config.yml

diff --git a/_config.yml b/_config.yml
index 61cc72d..957ae93 100644
--- a/_config.yml
+++ b/_config.yml
@@ -67,7 +67,7 @@ footer:
   copyright:
 
   # Powered by hexo & NexT
-  powered: true
+  powered: false    # 个人喜欢极简的风格，所以去掉主页底部的"Powered by hexo & NexT"
 
   # Beian ICP and gongan information for Chinese users. See: https://beian.miit.gov.cn, http://www.beian.gov.cn
   beian:
@@ -97,10 +97,10 @@ creative_commons:
 # ---------------------------------------------------------------
 
 # Schemes
-scheme: Muse
+#scheme: Muse
 #scheme: Mist
 #scheme: Pisces
-#scheme: Gemini
+scheme: Gemini     # 换一个带边栏的风格，scheme是NexT内部的风格
 
 # Dark Mode
 darkmode: false
@@ -117,10 +117,10 @@ darkmode: false
 # External url should start with http:// or https://
 menu:
   home: / || fa fa-home
-  #about: /about/ || fa fa-user
-  #tags: /tags/ || fa fa-tags
+  about: /about || fa fa-user    # 打开about, tags, archives标签
+  tags: /tags || fa fa-tags
   #categories: /categories/ || fa fa-th
-  archives: /archives/ || fa fa-archive
+  archives: /archives || fa fa-archive
   #schedule: /schedule/ || fa fa-calendar
   #sitemap: /sitemap.xml || fa fa-sitemap
   #commonweal: /404/ || fa fa-heartbeat
@@ -138,8 +138,8 @@ menu_settings:
 
 sidebar:
   # Sidebar Position.
-  position: left
-  #position: right
+  #position: left
+  position: right     # 边栏移动到右边
 
   # Manual define the sidebar width. If commented, will be default for:
   # Muse | Mist: 320
@@ -261,12 +261,12 @@ tag_icon: false
 # Front-matter variable (unsupport animation).
 reward_settings:
   # If true, reward will be displayed in every article by default.
-  enable: false
+  enable: true        # 打开打赏的功能
   animation: false
   #comment: Donate comment here.
 
 reward:
-  #wechatpay: /images/wechatpay.png
+  wechatpay: /images/weixinpay.svg       # 把微信的付款二维码放到这个地方
   #alipay: /images/alipay.png
   #paypal: /images/paypal.png
   #bitcoin: /images/bitcoin.png
@@ -354,7 +354,7 @@ codeblock:
   # Code Highlight theme
   # Available values: normal | night | night eighties | night blue | night bright | solarized | solarized dark | galactic
   # See: https://github.com/chriskempson/tomorrow-theme
-  highlight_theme: normal
+  highlight_theme: night blue            # 选择嵌入代码的显示风格，这里是深蓝色底
   # Add copy button on codeblock
   copy_button:
     enable: false
@@ -823,7 +823,7 @@ mermaid:
 # Use velocity to animate everything.
 # For more information: http://velocityjs.org
 motion:
-  enable: true
+  enable: false          # 不禁止这个的话，打开一个文章会有动画，个人不喜欢这个
   async: false
   transition:
     # Transition variants:
```

 * 添加文章和生成静态网页

   hexo n "文章名字"
   hexo clean && hexo g

   会在public目录里生成相关的静态网页，把public里的内容copy出来然后push到如上的
   username.github.io仓库里。

   本地调试的话可以先运行: hexo server, 然后在本地浏览器里通过http://localhost:4000
   观察生成的静态网页。

 * 保存本地改动到GitHub

   cd hexo/blog; git add <改动文件>; git commit -s -m <改动描述>;
   cd hexo/blog/themes/next; git add <改动文件>; git commit -s -m <改动描述>;
   git remote add url_priv_hexo
   git push --recurse-submodules=on-demand origin master:master

   在其他地方可以使用如下的方式迭代clone hexo和NexT仓库:
   git clone --recursive url_priv_hexo
