---
layout: single
author_profile: true
comments: true
title: expect 模拟自动登录过程
categories: [Linux]
tags: [Linux]
---

1、首先写一个模拟登录的脚本，需要用户输入用户名和密码进行交互

login.sh脚本

<pre class="lang:sh decode:true">#!/bin/sh
username="root"
passwd="root"
login=0
for k in $(seq 3)
do
	read -p "Username:" iusername
	read -p "Password:" ipasswd
	if [ "$username" = "$iusername" ] &amp;&amp; [ "$passwd" = "$ipasswd" ] 
	then
		echo "Welcome back. Root"
		login=1
		break
	fi
	echo "input error"
done
if [ $login -eq 0 ] 
then
	echo "login fail"
fi</pre>

2、用expect实现自动交互，自动输入用户名、密码进行登录.

apt-get install expect

whereis expect发现expect脚本的路径在/usr/bin/expect

spawn的功能是给ssh运行进程加个壳，用来传递交互指令

expect “Password:”是收到期望字符串之后，进行下一步

send “root\r” 是模拟用户输入，注意末尾的回车符

autologin.sh脚本
<pre class="lang:sh decode:true">#!/usr/bin/expect
spawn ./login.sh
expect "Username:"
send "root\r"
expect "Password:"
send "root\r"
interact</pre>
