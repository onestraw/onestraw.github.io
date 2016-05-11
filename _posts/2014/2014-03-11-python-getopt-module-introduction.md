---
layout: single
author_profile: true
comments: true
title: python getopt函数详解
categories: [Python]
tags: [Python]
---
getopt模块可以帮助脚本解析sys.argv的命令行参数，它遵守和Unix getopt()函数相同的约定，该模块提供两个函数和一个异常。

<strong>1. getopt.getopt(args, options[, long_options])</strong>

<strong>功能：</strong>解析命令选项和参数列表

<strong>args:</strong> 要解析的参数列表，传入命令行参数要传入sys.argv[1:]

<strong>options:</strong> 由打算获取的选项字母组成的字符串，要获得某个选项的值，需要在代表该选项的字母后加冒号’:’
如 -a 1000 -b 123 -d onestraw.net
1）如果想获得3个选项的值，options=”a:b:d”
2）此时，不能只获得后面选项的值，如”abd:”，这样是错误的
3）但是可以只获得前面选项的值，如”a:bd”，也可以用”a:b”，但不能是”a:”

<strong>有一点和Unix GNU getopt()函数，一个非选项参数后面，所有的参数都被认为是非选项参数</strong>

options = “a:bd:”和options = “a:bd”效果一样，都得不到选项d的值

如果参数列表为 -a -b 123 -d onestraw.net
如果要获得选项a的值，它不会返回null, 会返回-b
<pre>&gt;&gt;&gt; args = "-a 1000 -b 123 -d onestraw.net".split()
&gt;&gt;&gt; args
['-a', '1000', '-b', '123', '-d', 'onestraw.net']
&gt;&gt;&gt; getopt.getopt(args,'a:b:d:')
([('-a', '1000'), ('-b', '123'), ('-d', 'onestraw.net')], [])
&gt;&gt;&gt; getopt.getopt(args,'a:bd:')
([('-a', '1000'), ('-b', '')], ['123', '-d', 'onestraw.net'])
&gt;&gt;&gt; getopt.getopt(args,'a:bd')
([('-a', '1000'), ('-b', '')], ['123', '-d', 'onestraw.net'])
&gt;&gt;&gt; getopt.getopt(args,'ab:d')
([('-a', '')], ['1000', '-b', '123', '-d', 'onestraw.net'])
&gt;&gt;&gt;</pre>
<strong>long_options: </strong>是getopt函数的可选参数，它必须是一个列表

如果命令行输入参数
–condition=send –srcIP=10.10.10.10 –dstIP=20.20.20.20
(等价于 –condition send –srcIP 10.10.10.10 –dstIP 20.20.20.20)
想要获得这三个长选项的值
long_options=["condition=","srcIP=","dstIP="]
（注意，不需要写’–’，但要写上’='）

long_options会根据前缀最大匹配原则搜索参数列表，也就是说有时候在命令行中选项输入不完整也能正常传入参数
假设选项只有3个，输入
–condition=send –srcIP=10.10.10.10 –dstIP=20.20.20.20
和输入
–c=send –s=10.10.10.10 –d=20.20.20.20

使用 long_options=["condition=","srcIP=","dstIP="]得到的结果一样
（但是当结果不唯一时，就会报错，小心使用）
<pre>&gt;&gt;&gt; largs="--c=send --s=10.10.10.10 --d=20.20.20.20".split()
&gt;&gt;&gt; getopt.getopt(largs,'',["condition=","srcIP=","dstIP="])
([('--condition', 'send'), ('--srcIP', '10.10.10.10'), ('--dstIP', '20.20.20.20')], [])
&gt;&gt;&gt;</pre>
<strong>返回值：</strong>
getopt()函数的返回值是一个元组，包括两个list，第一个list是根据选项得到的如（option, value）这样的结果，第二个list是参数列表args解析剩余的部分

<strong>2. getopt.gnu_getopt(args, options[, long_options])</strong>

它和上面的getopt()一样，除去限制：一个非选项参数后面，所有的参数都被认为是非选项参数

<strong>3. exception getopt.GetoptError</strong>

当getopt()函数执行出错时，就会抛出该异常。

<strong>4. exception getopt.error</strong>

GetoptError的别名，为了向后兼容。

参考http://docs.python.org/2/library/getopt.html
