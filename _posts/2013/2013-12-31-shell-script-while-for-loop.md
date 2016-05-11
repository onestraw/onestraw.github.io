---
layout: single
author_profile: true
comments: true
title: shell脚本的while/for循环
categories: [Linux]
tags: [Linux]
---
任务：写一个shell脚本，调用nmap命令，实现对一个主机的循环扫描。

问题：shell脚本中如何使用while , for循环

## 一、使用while循环实现

{% highlight bash %}
#!/bin/sh
n=1
while [$n -le 10 ]
do 
	nmap -A -v 192.168.0.1
	let n+= 1
done

echo "scan done at".`date`

{% endhighlight %}

### 1. 比较语句

	-le less equal小于等于
	-lt less than小于
	-gt greater than大于
	-ge greater equal大于等于

如

	k=1
	while [ $k -le 10 ]

可以循环10次

### 2、赋值语句

1).let

	let k+=1
	let k-=1

2).expr

	k=`expr $k + 1 `
	k=`expr $k – 1 `

注意“是Tab键上面的反引号，作用是获得引号内的语句在终端的执行结果。   

3).1对方括号[]

	k=$[$k+1]

等号两边不能有任何空格  

4).两对圆括号()

	k=$(($k+1))

等号两边不能有任何空格

### 3、由于缺少空格引起的问题

1).如果第5行写成

	while [$n -le 10 ]

‘[' 和 $n之间没有空格
提示错误：

	./loopscan.sh: 行 5: [1: 未找到命令  

2).如果第5行写成

	while [ $n -le 10 ]

']'和 10 之间没有空格, 提示错误

	./loopscan.sh: 第 5 行: [: 缺少 `]‘

3).如果$k、1和减号之间没有空格

	k=’10-1′

可以在终端尝试

	k=10
	expr $k-1

使用man expr查看expr用法

## 二、使用for循环实现

### 1. 两对圆括号

	for((k=1;k&lt;=10;k++))
	do
	echo $k
	done

### 2. seq

	for k in $(seq 10)
	do
	echo $k
	done

seq是一个bash命令，可在终端用man seq查询用法

### 3. seq生成一个序列，用法如下

1).当后面只跟一个整数时，默认起始值为1, 步幅间隔为1

	seq 10

生成一个1-10的数列

2).指定序列的初始值

	seq 5 10
	
生成一个5,6,7,8,9,10的数列

3).指定序列的间隔

	seq 10 5 50
	
生成一个5,15,20,…,50的序列

4).可以负数，可以逆序

	seq 10 -2 -10

生成10,8,6,…,-6,-8,-10的序列
