---
layout: single
author_profile: true
comments: true
title: python pickle module
category: Python
tags: [Python]
---
<h1><strong>一、pickle模块概述</strong></h1>
<strong>pickle</strong>
这个单词的中文意思为
n. 腌菜，泡菜；腌制食品；
v. 腌渍

腌菜是为了长期保存蔬菜，在以前没有冰箱等冷藏技术的时代，一般在秋天腌制蔬菜以便在冬天吃，pickle模块的作用类似于腌菜的作用。

<strong>pickle模块</strong>实现了一个基本但强大的算法，用于对Python对象结构进行序列化（serializing）和反序列化（de-serializing）

<strong>序列化：</strong>它是一个将任意复杂的对象转成对象的文本或二进制表示的过程，对应pickle模块中的dump()函数，简单的说，就是将一个复杂的对象用简单的方法保存至磁盘文件，长期存储。
<strong>反序列化：</strong>将对象经过序列化后的形式恢复到原有的对象的过程，对应pickle模块中的load()函数，从硬盘文件恢复原来的数据对象。

<strong>cPickle模块</strong>是pickle模块的一个优化版本，cPickle使用C语言实现，它比pickle快1000倍，cPickle和pickle有微小的差别，具体请查看文档。
<h1><strong>二、cPickle操作函数</strong></h1>
<tt>1. </tt><tt>dump</tt><big>(</big><em>obj</em>, <em>file</em>[, <em>protocol</em>]<big>)</big>

obj: 输出到文件的数据对象
file: 已打开的文件句柄（描述符），并且此file可写
protocol: 默认是0，表示以文本的形式写入file。protocol的值还可以是负数，表示以二进制的形式写入file。

dump()函数会在pickle文件中标识每次写入对象的数据类型，两次dump()操作写入的对象在文件中有特定的分隔符来标识，通过查看写入的pickle文件，分隔符好像是 ‘.’

2. <tt>load</tt><big>(</big><em>file</em><big>)</big>

file:  file: 已打开的文件句柄（描述符）

load()返回的对象是dump()一次写入的对象，load函数从pickle文件顺序读取。

<tt>3.dumps</tt><big>(</big><em>obj</em>[, <em>protocol</em>]<big>)</big>

参数含义同dump()，不同的是该函数不写入文件，而是返回一个string对象

<tt>4.loads</tt><big>(</big><em>string</em><big>)</big>

从dumps()返回的string对象获得原始的数据
<h1><strong>三、测试cPickle</strong></h1>
<pre>import cPickle,pprint

class Student():
	def __init__(self,sno="0", name="sword", age=20):
		self.sno = sno
		self.name = name
		self.age = age
	def printStudent(self):
		print("sno:%s\nname:%s\nage:%d\n" %(self.sno,self.name,self.age))
#dump
print("----dump----")
fp = open("data.pk","wb")
x=[1,[2,[3,[4,'a','b']]]]
cPickle.dump(x,fp)
for i in range(3):
	cPickle.dump("hello,pickle",fp)
	cPickle.dump(i,fp)

stu = Student("2014001","geekSword",23)
cPickle.dump(stu, fp)
fp.close()
#load
print("----load----")
fp = open("data.pk","rb")
x=cPickle.load(fp)
pprint.pprint(x)
for i in range(3):
	print(cPickle.load(fp))
	print(cPickle.load(fp))
	
s = Student()
s.printStudent()
s = cPickle.load(fp)
s.printStudent()
fp.close()

#dumps
print("----dumps &amp; loads----")
str = cPickle.dumps(stu)
print(str)
s = cPickle.loads(str)
s.printStudent()</pre>
<h1>四、pickle文件分析</h1>
(lp1
I1
a(lp2
I2
a(lp3
I3
a(lp4
I4
aS’a’
aS’b’
aaaa.

这是列表x=[1,[2,[3,[4,'a','b']]]]在pk文件中的形式，猜测各种符号的含义:

lp[1-4] 表示列表的深度，首字母’l'表明这是一个list对象
I 表示后面的内容为整数aS 表示后面的内容为字符
a(  和  a 表示一对 []
.  最后的 . 表示一次dump操作的结束（一个数据对象的结束，分隔）

S’hello,pickle’

这是dump(“hello,pickle”, fp)结果在pk文件中的形式
大S表示后面的内容是一个string对象

(i__main__
Student
p1
(dp2
S’sno’
p3
S’2014001′
p4
sS’age’
p5
I23
sS’name’
p6
S’geekSword’
p7
sb.

这是自定义对象stu在pk文件中的形式。
