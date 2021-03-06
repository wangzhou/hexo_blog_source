---
title: 使用github做开源开发
tags:
  - 软件开发
description: 对于一个使用github做开发的开源项目，本文梳理其中的基本逻辑。
abbrlink: 96a1d14c
date: 2021-06-19 10:27:46
---

一个项目的参与者有开发者和维护者。开发者要做的是，从主分支拉代码，开发，提交代码。
维护者要做的是，review开发者提交的代码，合入代码。

开发者在github上使用发git pull request的方法向维护者提交合入请求。
```
                      repo fork
  +-----------------+ -------> +-------------+
  | 主线repo master |          | 开发者 repo |
  +-----------------+ <------- +-------------+
           \      git pull request    ^
            \                          \
             \                          \ git push dev-branch到开发者github repo
              \                          \
               \                          \
                \                          \
                 \             +----------------+
                  ---------->  | 开发者本地repo |
             跟踪主线master    +----------------+
                           依据最新master分支创建开发分支: dev-branch
```
如上，开发者本地可以维护一个主线repo master的跟踪分支，开发一个新特性的时候，
开发者建立基于最新主线master的开发分支，开发完成后，把开发分支push到开发者github
repo上，然后再在github页面上发起git pull request。

维护者在收到git pull request的时候，可以在线review，也可以把需要review的patch
拉到本地。维护者可以用 git pull origin pull/<pull_request_id>/head把对应patch拉到
本地的当前分支上。维护者直接在github页面就可以点击合入补丁，合入补丁的时候有几种
方式，"Create a merge commit"的方式会在合入的时候自动创建一个merge的commit，记录
整个merge的信息; "Rebase and merge"的方式会只合入pull request里的patch。
