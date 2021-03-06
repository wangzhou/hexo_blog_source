---
title: git使用笔记
tags:
  - 软件开发
  - git
description: 这篇文档的开始是got git这本书的读书笔记。后面开始是一些git使用技巧的总结。
categories: read
abbrlink: a722b34f
date: 2021-07-17 11:00:34
---

chaper 2
---------

git add -u 加入缓冲区所有的改动文件
        -A 把所有增加/删除的文件加入缓冲区
	-i 提供一个添加文件的交互界面

git diff 显示改动，只显示没有提交到缓冲区的改动
         --cached 显示缓冲区中的改动
	 old-ID new-ID 显示两次提交的改动

git stash/git stash pop 工作的时候往往要临时切换到一个分支，这时可以用git stash
	不需要git add, 当处理完另一个分支时，切换会原来的分支，使用git stash pop
	即可以回到原来的工作区

chapter 3
-----------

 中文：UTF-8字符集(一般)

chapter 4
---------

 git config -e           改当前库的配置(.git/config)
 git config -e --global  改当前用户的配置(/home/***/.gitconfig)
 git config -e --system  改整个系统的配置(/etc/gitconfig)

chapter 5
---------

 git log --oneline 精简模式显示提交信息
 git log --prety=fuller 显示AuthorData和Commit Date ?
 git ls-tree -l HEAD 查看版本库中的最新提交
 git ls-files 查看缓冲区的最新内容

chapter 6
---------

 git cat-file -t <ID> 显示ID对应的类型(type)：commit, tree, blob...
              -p <ID> 显示ID对应的内容
 git rev-parse HEAD/master... 显示对应的ID
 git log --graph --all 显示commit的历史，加上--all可以显示所有分支
 
 HEAD: 保存当前分支的路径，若当前分支test, HEAD内容是 ref: refs/heads/test
 .git/refs/heads/master : heads目录下存放所有的分支，若此时git中还有一个分支test
                          在heads下会发现：master，test. master, test文件中放的
			  是ID，是对应分支最新提交的ID
 .git/refs/tags/*** : ...

chapter 7
---------

 git reset --hard HEAD^ 整个HEAD切换到他的父提交, 若现在有如下的提交:
 A-->B-->C-->D 
 使用上述命令后，使用git log将只看到: A-->B-->C
 git reset --hard 将版本库，缓存区，工作区全部切换到相应的版本(C), D版本相当于
 被丢掉了(没有显示出来)

 git reset --soft HEAD^ 只把版本库切到了父提交，也就是回到了，上次git add ***
 git commit *** 之前的状态，使用git status可以证明这点

 git reset HEAD^ 把版本库，缓存区切到了父提交，也就是回到了上次编辑过工作区，
 git add *** 之前的状态，使用git status可以看到这点

 git reset/git reset HEAD 依照上面的分析，相当于缓冲区切到父提交，就是把git add
 加入缓冲区的东西去掉，是git add 的逆操作

 git reset -- filename 是git add filename的逆操作

 git reset --hard HEAD^ 之后的挽救措施：（想恢复原来的提交）
 在.git/log/logs/HEAD中记录着每次HEAD的改动，找到想要的ID用来恢复
 更简单的方法：
 git reflog show 找到要恢复的版本
 git reset --hard master@{***} 即可

 注: git reset --hard HEAD^
     do some change... 
     git add ...
     git commit ... (version E)
     实际是存储是：(其中version D是不可见的)
```
     A-->B-->C-->D
              \-->E
```
     git reset 没有改变HEAD的内容，而是改变了.git/refs/head/... 的内容

chapter 8
---------

 git checkout ID 检出ID所对应的提交. 比如；
 A-->B-->C-->D 
 git checkout ID(C) ID(C)表示C对应的ID
 这是用git branch察看所在的分支，会显示当前处于no branch的状态，实际上察看
 .git/HEAD会发现其中的内容不是指向一个分支(如：ref:refs/heads/master), 而是一个
 具体提交的ID. 在这种no branch的状态可以查看代码，做验证，但是不能提交修改。
 其实也是可以在提交的，只是再从当前的状态切回某个分支(如：git checkout master),
 之前的提交不可见了:
```
               /-- master
 A-->B-->C-->D 
      \
       E -- git checkout ID(B), git commit E
```
 如上在no branch上提交了E，然后git checkout master切回了master这时候E不可见了.
 用git reflog show 查看提交的历史，然后git reset --hard HEAD@{...} 可以把HEAD
 指向E，这时 A-->B-->E 成了master分支，C、D不可见了
 
 git checkout -b branch_name 创建新的分支，名字是branch_name
 git checkout 改变HEAD的内容

chapter 12
------------

 git cherry-pick id 把id对应的commit向当前的HEAD提交
 A-->B-->C-->D-->E git checkout id(C) git cherry-pich id(E)会把E向C提交:
```
                  /-- master
 A-->B-->C-->D-->E 
          \
           E <--HEAD
```
 这时HEAD分离的情况，HEAD不对应任何分支,可以建立新的branch, 也可以git reset可以
 把master的内容指向E, 这时D和其后的E将显示不出来

 git cherry-pick id -e 可以修改commit的签名中的内容(邮箱)

 git rebase

chapter 15
------------

 git pull/push
 git push 有时无法成功，可能是因为git push对应的git仓库不是bare的，直接推送会
 改变工作区。这可以配置对应的远程仓库：git config receive.denyCurrentBranch ignore
 这时可以成功push

 chapter 16
------------

 -other
 git commit --amend --author='your name <email-box>' 可以修改commit中author一行的内容

 patch的subject这一行有时不只是[PATCH], 比如在询问意见时可以是[PATCH RFC ***], 在第3版
 patch时subject可以是[PATCH v3 ***]. 如何改变subject这一行的内容：可以在生成patch
 的时候加--subject-prefix="***", 比如, git format-patch -s -2 --subject-prefix="PATCH RFC"
 生成的patch subject为：[PATCH RFC 0/3], [PATCH RFC 1/3], [PATCH RFC 2/3], [PATCH RFC 3/3]

 git send-email 使用git send-email发送patches, 组成的patches是一个系列的。
 用git format-patch生成patches, 然后一个个用普通邮箱发出，给出的patches是一个个分裂的。
 git send-email *.patch 即可把当前目录里的patch都发送出去，而且git send-email提供一个
 对话是的发送过程，只要在过程中填入发送的邮箱即可。对于cc的邮箱可以在一开始的命令中给出：
 git send-email *.patch --cc=your_email_box@126.com

 Message-ID to be used as In-Reply-To?


git pull note
--------------

1. git repo A:
   branch: master, test

   git repo B;
   branch: master, test(all pull from repo A)

   若在repo A上test分支加一个提交, 在repo B的master分支上用git pull, reop
   B的test分支将不会更新，repo B切换到test分支上，再使用git pull,
   则可以更新test分支。

2. 还是上面的场景，在repo A test分支上加一个commit new。在repo B中git fetch,
   git checkout origin/test, git log, 会发现现在repo B的远程分支origin/test
   有了repo A test分支上的commit new

   在repo B的test分支上，git merge orgin/test，即可把git fetch得到的repo B
   origin/test分支和test合并。这也就是常说的git pull = git fetch + git merge

   可以看出repo B在本地是有origin/master, origin/test的远程分支的完整拷贝，也有
   本地分支master, test。在git fetch操作时，只是把repo A上的新提交加到repo B的
   origin/test“分支”上。
```
                       git clone
   repo A: A-->B-->C   ========>   repo B: A-->B-->C 
                   \                                    \
                  master                           master(也是origin/master)

                        new commit
                       /  
   repo A: A-->B-->C-->D           repo B: A-->B-->C 
                       \                            \
                        master                       master(也是origin/master)

                        new commit        git fetch    origin/master
                       /                               / 
   repo A: A-->B-->C-->D           repo B: A-->B-->C-->D
                       \                            \
                        master                      master

                        new commit       git merge     origin/master
                       /                               / 
   repo A: A-->B-->C-->D           repo B: A-->B-->C-->D
                       \                                \
                            master                                master

   如果在第三步中在repo B的master分支上又作了几次提交,比如:
                        new commit                     origin/master
                       /                               / 
   repo A: A-->B-->C-->D           repo B: A-->B-->C-->D
                       \                            \
                       master                        -->E-->F  master
                      
   那么在git merge会如下, master分支中会加入E, F两个提交
                        new commit        git merge    origin/master
                       /                               / 
   repo A: A-->B-->C-->D           repo B: A-->B-->C-->D-------
                       \                            \          \
                       master                        -->E-->F-->G  master
```
3. git branch显示本地分支，git branch -r显示远程分支，
   git checkout orgin/test -b local_branch 建立一个本地分支local_branch跟踪
   远程分支

4. 在repo B .git中的config文件中有这样的配置条目:
```
    [remote "origin"]
	    url = /home/example/git_test_client/../git_test
	    fetch = +refs/heads/*:refs/remotes/origin/*
    [branch "master"]
	    remote = origin
	    merge = refs/heads/master
    [branch "test_client"]
	    remote = origin
	    merge = refs/heads/test
```
    其中第一条[remote "origin"], url表示远程仓库的url, fetch表示做git fetch
    的时候远程仓库中的各个分支，对应本地仓库中的refs/remotes/orgin/下的各个“分支”。
    本地仓库的origin/master等严格的讲并不是一个分支，使用git checkout origin/master
    会显示处于头指针分离状态。
    后面的[branch "master"]条目表示，当时候git pull时，会把git fetch得到的orgin/master
    merge到本地的master分支中。

给branch添加描述信息
--------------------

   git branch --edit-description 可以添加当前git分支的说明，说明文字被添加到.git/config
   所以可以用git config -l 查看当前分支的说明，如果有说明的话。

一些开发中有用的小技巧
----------------------

 1. 已经知道了可以下载代码的git服务器的地址，比如：git://git.linaro.org/kernel.git
    可以使用：
        git clone git://git.linaro.org/kernel.git
    下在代码

    要是你的本地电脑上已经有了之前clone的一个kernel的git仓库，可以使用：
        git clone git://git.linaro.org/kernel.git --reference /path/to/kernel.git
    提高下载的速度，新的下载的git库将会重用以前已有的git库
        
    下载好git库后，可以使用：
        git branch -r 
    显示所用的远程分支, 然后用：
        git checkout branchname
    提取出需要的分支，然后用；
        git branch
    就可以看到上面提出来的分支的名字了

2. 显示远程仓库的网址和名字：
       git remote -v
   修改远程仓库，发现远程仓库的地址错了，导致一直下载不下来代码，需要添加正确
   的地址：
       git remote add hilt ssh://git.linaro.org/kernel.git
   其中hilt是这个远程仓库的名字
       git remote rm origin
   其中orgin是原来错误远程仓库的名字，之后把远程仓库的名字从hilt改成origin：
       git remote rename hilt origin

3. 在开发的时候会出现很多中间版本，这些版本做的改动对别人是无意义的, 比如有
   version_1--->version_2--->version_3, 怎么把这三个版本合并成一个版本(一次提交):
       git rebase -i HEAD~3
   其中3表示把最近的3次提交合并成一次提交
   
   如果commit的log message写的不好，也可以用：
       git commit --amend
   重写commit的log message

4. 代码改好了，需要制作patch，可以使用：
       git format-patch -s -1
   其中s表示patch中会加上签名项, 1表示对最近一次提交生成patch. 如果把1变成2，那么
   会生成两个patch, 以version_1--->version_2--->version_3为例，这两个patch是
   version_3对version_2的patch、version_2对version_1的patch
