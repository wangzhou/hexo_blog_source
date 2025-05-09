---
title: git submodule的使用
tags:
  - git
description: 本文介绍git中submodule的基本逻辑以及使用方法。
abbrlink: 46101
date: 2023-01-15 15:50:29
categories:
---

基本逻辑
---------

 当一个代码仓库以代码的方式完整包含另一个仓库的代码时，我们就可以使用git中submodule
 的方式来组织代码仓库，其中，被包含的仓库叫submodule/subproject, 包含他的仓库叫
 superproject，所谓submodule的配置都是针对superproject的。

 qemu就是用submodule的方式管理各种firmware的，比如，qemu把opensbi就作为它的一个
 submodule。

```
                                +-----------+
                                | project_a |
                                +-----------+
                                 /
                                /
    +-----------------------+  /
    | superproject          | /
    |   |                   |/
    |   +----               |
    |   |                  /|
    |   +----             / |
    |   |                /  |
    |   +----- ./project_a  |
    +-----------------------+
```
 如上示意图，superproject和project_a各自都是独立的git仓库。所谓superproject把
 project_a作为一个子仓库，只是在superproject中加入project_a的配置信息，比如，project_a
 的URL/branch/commit，这些信息可以确定作为superproject子仓库的project_a的一个精确
 的提交。

 同时superproject中会有一个目录存放project_a的代码。

使用示例
---------

 我们下面具体看怎么把一个独立的仓库作为子仓库添加给superproject。

```
sherlock@m1:~/tests/git_submodule/superproject$ git submodule add https://github.com/wangzhou/notes  submodule/notes
Cloning into '/home/sherlock/tests/git_submodule/superproject/submodule/notes'...
remote: Enumerating objects: 1397, done.
remote: Counting objects: 100% (120/120), done.
remote: Compressing objects: 100% (86/86), done.
remote: Total 1397 (delta 68), reused 81 (delta 34), pack-reused 1277
Receiving objects: 100% (1397/1397), 5.82 MiB | 852.00 KiB/s, done.
Resolving deltas: 100% (680/680), done.
sherlock@m1:~/tests/git_submodule/superproject$ 
```
 我们这里把名字叫notes的这个仓库(https://github.com/wangzhou/notes)作为superproject
 的一个子仓库。

 如上添加命令后，我们可以看到superproject的目录下多了.gitmodules文件，submodule/notes
 目录里也多了notes仓库的代码。当我给superproject加notes这个子仓库的时候，git顺着
 notes的URL找到notes，并把notes的当前分支上的最新提交作为子仓库的检出点。
```
sherlock@m1:~/tests/git_submodule/superproject$ cat .gitmodules 
[submodule "submodule/notes"]
        path = submodule/notes
        url = https://github.com/wangzhou/notes
sherlock@m1:~/tests/git_submodule/superproject$ git submodule status
 e78b0444f95c05e6cc9a2aa953dfd135e6cde1ef submodule/notes (heads/master)
```

 当我们把.gitmodules和submodule/notes提交superproject后，会发现submodule/notes
 的改动就一行，它指向子仓库的一个提交。
```
sherlock@m1:~/tests/git_submodule/superproject$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   .gitmodules
        new file:   submodule/notes

sherlock@m1:~/tests/git_submodule/superproject$ git add .gitmodules submodule/notes/
sherlock@m1:~/tests/git_submodule/superproject$ git commit -s -m "add note as a submodule"
[master 95fe141] add note as a submodule
 2 files changed, 4 insertions(+)
 create mode 100644 .gitmodules
 create mode 160000 submodule/notes

sherlock@m1:~/tests/git_submodule/superproject$ git log -p
[...]
diff --git a/.gitmodules b/.gitmodules
new file mode 100644
index 0000000..5f6b923
--- /dev/null
+++ b/.gitmodules
@@ -0,0 +1,3 @@
+[submodule "submodule/notes"]
+       path = submodule/notes
+       url = https://github.com/wangzhou/notes
diff --git a/submodule/notes b/submodule/notes
new file mode 160000
index 0000000..e78b044
--- /dev/null
+++ b/submodule/notes
@@ -0,0 +1 @@
+Subproject commit e78b0444f95c05e6cc9a2aa953dfd135e6cde1ef
```

git submodule里有和子仓库相关的各种自命令，比如，我们可以用如下的命令更新子仓库，
可以看见superproject的子仓库的提交也更新了。
```
sherlock@m1:~/tests/git_submodule/superproject$ git submodule update --remote 
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (1/1), done.
remote: Total 3 (delta 2), reused 3 (delta 2), pack-reused 0
Unpacking objects: 100% (3/3), 289 bytes | 96.00 KiB/s, done.
From https://github.com/wangzhou/notes
   9ed1125..a24c0b1  master     -> origin/master
Submodule path 'submodule/notes': checked out 'a24c0b112dd342e2c00ec79dfe4f5f3fbfcd09b4'
sherlock@m1:~/tests/git_submodule/superproject$ git diff
diff --git a/submodule/notes b/submodule/notes
index e78b044..a24c0b1 160000
--- a/submodule/notes
+++ b/submodule/notes
@@ -1 +1 @@
-Subproject commit e78b0444f95c05e6cc9a2aa953dfd135e6cde1ef
+Subproject commit a24c0b112dd342e2c00ec79dfe4f5f3fbfcd09b4
```
