---
title: git am 冲突解决技巧
tags:
  - git
description: 本文记录一种遇到git am冲突时的解决办法
abbrlink: 3c148803
date: 2021-07-05 22:34:58
categories:
---

使用git am合patch的时候可能有冲突出现，这个时候，手动解决的办法是看看冲突在哪里，
然后手动的把那个patch和入。手动合入需要的时间太长.

我们可以用git apply --reject patch的方式合入。这里需要注意几个问题。

git apply只会看到文件，它把patch里的一个个diff段拆出来, 然后合入相应的文件里,
而且git apply只会合入当前目录下的diff段，所以上面的命令要到所有diff段的最大的
一个目录里去执行，一般为了方便就在代码的根目录里执行。git apply后相当于修改了
原文见，所以要git add，git commit下。--reject的这个参数会把有冲突的段保存在一个
.rej的文件里。

所以，一般git am合patch的步骤可以是这样的：

1. git am patch     --> 没有conflict，over！

2. 有冲突的时候： cd code_root/
		  git apply --reject patch

3. 在.rej文件里找见冲突的diff段，手动修改对应的代码

4. git add related_files

5. git am --resolved

注意最后一个操作, 我们现在已经把git am的冲突解决，用git am --resovled可以继续git
am的操作把commit log也自动的打上！
