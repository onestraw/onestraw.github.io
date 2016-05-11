---
layout: single
author_profile: true
comments: true
title: lambda、filter、map、reduce函数
category: Python
tags: [Python]
---
<strong>1. lambda函数</strong>

python中的lambda函数借鉴自LISP，是一个匿名函数(是函数，又不像def定义的那样有函数名)，使用形式如下：

lambda 参数列表:函数体

参数列表可以为空，多个参数用逗号’,'分隔

<pre class="lang:python decode:true">&gt;&gt;&gt; (lambda :2*3)()
6
&gt;&gt;&gt; (lambda x:x%10)(123)
3
&gt;&gt;&gt; f = (lambda x,y,z: x*2+y*3+z*4)
&gt;&gt;&gt; f(1,2,3)
20
&gt;&gt;&gt;</pre>

<strong>2.filter函数</strong>

filter(function, iterable)

顾名思义，filter函数用第一个参数function函数对iterable进行过滤（function函数的返回值是一个布尔值），返回一个和iterable同类型的数据，iterable可以是字符串、列表、字典等

&nbsp;

<pre class="lang:python decode:true">&gt;&gt;&gt; a=range(1,20,2)
&gt;&gt;&gt; a
[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
&gt;&gt;&gt; filter((lambda x:x%3==0),a)
[3, 9, 15]
&gt;&gt;&gt;</pre>

<strong>3.map函数</strong>

map(function, iterable, …)

对iterable中的每个元素执行function函数，如果iterable不止一个，那么function函数的参数个数和iterable个数一致，并且各个iterable元素个数不同时，会被扩展成None item

<pre class="lang:python decode:true">&gt;&gt;&gt; a=range(1,20,2)
&gt;&gt;&gt; a
[1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
&gt;&gt;&gt; map((lambda x: x**2),a)
[1, 9, 25, 49, 81, 121, 169, 225, 289, 361]
&gt;&gt;&gt; b=range(2,20,2)
&gt;&gt;&gt; b
[2, 4, 6, 8, 10, 12, 14, 16, 18]
&gt;&gt;&gt; map((lambda x,y:x+y),a,b)

Traceback (most recent call last):
  File "&lt;pyshell#48&gt;", line 1, in &lt;module&gt;
    map((lambda x,y:x+y),a,b)
  File "&lt;pyshell#48&gt;", line 1, in &lt;lambda&gt;
    map((lambda x,y:x+y),a,b)
TypeError: unsupported operand type(s) for +: 'int' and 'NoneType'
&gt;&gt;&gt; b.append(100)
&gt;&gt;&gt; map((lambda x,y:x+y),a,b)
[3, 7, 11, 15, 19, 23, 27, 31, 35, 119]
&gt;&gt;&gt;</pre>

<strong>4.reduce函数</strong>

reduce(function, iterable[, initializer])

对iterable中的item顺序迭代调用function，如果有initializer，还可以作为初始值调用

python文档中给出一个基本等价的实现

<pre>def reduce(function, iterable, initializer=None):
    it = iter(iterable)
    if initializer is None:
        try:
            initializer = next(it)
        except StopIteration:
            raise TypeError('reduce() of empty sequence with no initial value')
    accum_value = initializer
    for x in it:
        accum_value = function(accum_value, x)
    return accum_value</pre>
    
&nbsp;

<pre class="lang:python decode:true ">&gt;&gt;&gt; a=range(1,101)
&gt;&gt;&gt; reduce((lambda x,y: x+y),a)
5050
&gt;&gt;&gt; reduce((lambda x,y: x+y),a,10000)
15050
&gt;&gt;&gt;</pre>
