---
title: tmux tips
tags:
  - 开发工具
  - tmux
description: >-
  tmux is useful tool in linux when you need to work in different windows. This
  note tries to collect and share some tips when I am using tmux to work.
abbrlink: dd20f22d
date: 2021-07-05 22:23:32
categories:
---

There are three basic concepts in tmux: pane, window, session.
After you run tmux in shell, you open a pane in a window in a session.

Let's first consider the pane operation in one window:

	ctl + B + %: splite current pane into two, left and right
	ctl + B + ": splite current pane into two, up and down
	ctl + B + z: full screen for current pane, or return back.

If you want multiple window:

	ctl + B + c: create a window

If you login your linux system, you can:

	tmux ls: to see the sessions	
	tmux a -t session_name(or session number): to attach to related session

If you want to copy log in tmux to a file:

	ctl + b + :
	capture-pane -S -3000 + return  this copied 3000 lines into buffer
	ctl + b + :
	save-buffer /path/to/your_file  this copied content in buffer to file,
	                                in my environment, path should be a full path.

tmux will use .bash_profile by default, so if you want to use the configures in
.bashrc，you should "source ~/.bashrc" in .bash_profile. One problem I met is tmux
fails to output colors for some commands such as ls, "source ~/.bashrc" in .bash_profile
can solve this problem.
