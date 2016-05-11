---
layout: single
author_profile: true
comments: true
title: 搭建Git服务器
tagline: 
category: essay
tags : [git]
---

只要在本地安装了git包，搭建Git服务器并不需要再安装额外的软件，为了简单，本文没有进行用户权限设置，
下面对使用github.com和使用本地git服务器创建新项目的步骤进行对比。

| 创建新repository步骤 |Github| 本地Git Server|
|:-----|:-----|:-----|
| 服务器 | https://github.com | 192.168.6.136|
| 帐户 | onestraw | root|
| 仓库路径 | https://github.com/onestraw | 192.168.6.136:/git |
| 在服务器上创建新仓库sample| 在github网站上Create New repository: sample| (in /git) git init --bare sample.git |
| |||
| git客户机（以下均在客户机上执行）| 192.168.6.1 |192.168.6.1 |
| 本地仓库初始化 | git init | git init |
| 添加到服务器分支 | git remote add origin https://github.com/onestraw/sample.git | git remote add origin 192.168.6.136:/git/sample.git |
| git add| git add hello.c |git add hello.c|
| git commit |git commit -m "add hello.c" |git commit -m "add hello.c"|
| git push | git push -u origin master |git push -u origin master|
| 查看服务器地址| git remote -v | git remote -v|

整体思路是这样的，当然也可以用下面这个[方法](https://git-scm.com/book/zh/v1/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%9C%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E9%83%A8%E7%BD%B2-Git)

###安装web界面

- [GitWeb](https://git-scm.com/book/zh/v1/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-GitWeb)


by geeksword
