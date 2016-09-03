---
layout: single
author_profile: true
comments: true
title: Find one path between two functioins
excerpt: "源码阅读辅助小工具"
tagline: 
category: Python
tags : [Python, Linux]
---

有许多优秀的源代码分析工具，如cscope, ctags和SI等，利用这些工具可以在源码中穿梭，可以查找符号，搜索函数定义，搜索被哪些函数调用，查找字符串等等，但是如果你知道两个函数，怎么找到它们之间的调用路径，[CodeViz](http://onestraw.net/cprogram/codeviz)可以生成函数调用图，但是CodeViz安装复杂，而且需要用修改过的gcc编译源码，虽然能输出可视化的图片，但是调用分枝太庞大，包含很多无用信息。

下面从分析cscope.out文件格式，构造函数调用关系图，DFS搜索两个函数之间的调用路径，针对Linux这样的大型项目，首先找出调用频率TOP 1000的函数，不作考虑，以提升搜索速度。

cscope 的四个参数

      -b       Build the database only.
      -c       Use  only  ASCII characters in the database file, that is, do not compress the data.
      -k       Kernel  Mode  turns  off  the  use of the default include dir (usually /usr/include) when  building the database, since kernel source trees generally do not use it.
      -R      Recurse subdirectories during search  for  source files.

cscope.out fromat

	A mark is a tab followed by one of these characters:

        Char   Meaning

        @      file
        $      function definition
        `      function call


reference
http://goo.gl/R2ErDs 

运行gengraph.sh生成 graph.out文件

{% highlight bash %}
#!/bin/bash
rm cscope.out
cscope -b -c -k -R
grep -e "[\$\`]" cscope.out > graph.out
sed -i 's/^[[:space:]]*//' graph.out
{% endhighlight %}

graph.out 文件的格式

	$函数名（定义，不是声明，不是被调用）
	`函数名（被调用）
	
例如Linux内核生成的graph.out 

	$vfs_read
	`rw_verify_area
	`do_sync_read
	`fsnotify_access
	`add_rchar
	`inc_syscr
	$do_sync_write
	`init_sync_kiocb
	`aio_write
	`wait_on_sync_kiocb

在图中，`$`后的函数是图的顶点，"`" 是前一个顶点的邻接顶点。

写完第一个版本，搜索小项目毫无压力，即使用递归的DFS也不会导致函数栈崩溃。但是像Linux这样的工程，由于没有压缩，cscope.out 有347M（linux3.12），graph.out也有36M，分析下发现代码中有很多printk,memcpy等函数调用，可认为树的叶节点，有什么方法找出所有的叶节点？ 统计！找出调用频率top 100（最后还是低估Linux了，top 5000中每个函数调用次数都在50以上）。  排除top 100后，图依然很大，先工作起来吧。

xroute.py

{% highlight python %}
#!/usr/bin/python
'''
find one path between two functions.
cannot process function pointer.
'''

import sys
import os
import operator

graph = "graph.out"
graph_lite = "graph-lite.out"
data = []
route = []
visited = []

def reverse_output_route(call):
	if call[1] >=0 :
		reverse_output_route(route[call[1]])
	print "\t->", call[0]

def non_recursive_dfs(f1, f2):
	global route 
	global visited 
	route = []
	visited = []
	stack = [(f1, -1)]
	while len(stack) > 0:
		call = stack.pop()
		if call[0] == f2:
			reverse_output_route(call)
			return True
		parent = len(route) 
		route.append(call)
		visited.append(call[0])
		try:
			no = data.index("$" + call[0]) + 1
			dedup = set()
			for i in range(no, len(data)):
				v = data[i]
				if v[0]!='`':
					break
				v = v[1:]
				if v not in visited and v not in dedup:
					dedup.add(v)
					stack.append((v, parent))
		except ValueError:
			pass

def top(n):
	f = open(graph, "r")
	data = f.read().splitlines()
	f.close()
	
	stat = dict()

	for name in data:
		k = name[1:]
		if k in stat:
			stat[k] += 1
		else:
			stat[k] = 1

	sorted_stat = sorted(stat.items(), key=operator.itemgetter(1), reverse=True)

	topdata = []
	i = 1 
	for k in sorted_stat:
		topdata.append(k[0])
		i+=1
		if i > n:
			break
	return topdata

def readdata():
	global data
	if os.path.isfile(graph_lite):
		fd = open(graph_lite, "r")
		data = fd.read().splitlines()
		fd.close()
		return

	fd = open(graph, "r")
	temp = fd.read().splitlines()
	fd.close()

	topdata = top(5000)
	i=1
	sz = len(temp)
	while i < sz:
		if temp[i][1:] not in topdata:
			data.append(temp[i])
			i += 1
			continue
		if temp[i][0] == '`':
			i += 1
		else:
			i += 1
			while i < sz:
				if temp[i][0] == '$':
					break;
				i += 1
	fd = open(graph_lite, "w")
	fd.write("\n".join(data))
	fd.close()

def main(f1, f2):
	readdata()
	#if not non_recursive_dfs(f1, f2) and not non_recursive_dfs(f2, f1):
	if not non_recursive_dfs(f1, f2):
		print "\nthere is no path between %s and %s\n" %(f1, f2)

def usage(pname):
	print "\n", pname, ": output the path between two functions\n"
	print "usage: ./%s [function a] [function b]\n" %(pname)
	sys.exit(-1)

if __name__ == '__main__':
	if len(sys.argv) < 3:
		usage(sys.argv[0])
	main(sys.argv[1], sys.argv[2])

{% endhighlight %}

示例

	root@ubuntu:~/linux-3.12# time ./xroute.py do_sys_open security_file_alloc
		-> do_sys_open
		-> do_filp_open
		-> path_openat
		-> get_empty_filp
		-> security_file_alloc

	real	0m25.051s
	user	0m24.799s
	sys	0m0.260s

			
缺点是比较明显，速度慢，不能处理函数指针。

顺便列出Linux3.12 中top 10的函数

    root@ubuntu:~# ./stats.py stats.out 10
    rank	function                      	frequency
    1	printk                        	44492
    2	kfree                         	24554
    3	dev_err                       	20570
    4	memcpy                        	18013
    5	spin_unlock_irqrestore        	16125
    6	mutex_unlock                  	15917
    7	spin_lock_irqsave             	13696
    8	memset                        	13518
    9	dev_dbg                       	12451
    10	BIT                           	12117
	
