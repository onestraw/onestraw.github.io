---
layout: single
author_profile: true
comments: true
title: 一个编程技巧
excerpt: python threading.Timer() 
categories: [Python]
tags: [Python]
---

#问题

规律性的办一件事，如每1小时查询一次更新，每10分钟写一次日志……总结为一句话：每隔一段时间调用一个函数。

#技巧
在看w3af源码时发现一个编程技巧，先看代码

	#ref：w3af/core/controllers/profiling/utils.py

	def dump_data_every_thread(func, delay_minutes, save_thread_ptr):
		"""
		This is a thread target which every X minutes
		"""
		func()

		save_thread = threading.Timer(delay_minutes * 60,
			dump_data_every_thread,
			args=(func, delay_minutes, save_thread_ptr))
		save_thread.start()

		if save_thread_ptr:
			# Remove the old one if it exists in the ptr
			save_thread_ptr.pop(0)

		save_thread_ptr.append(save_thread)

在dump_data_every_thread函数中，创建线程时，又使用了自身作为参数。
它实现的功能为：创建独立线程，每隔delay_minutes分钟执行一次func函数。

###Timer类原型

	class threading.Timer(interval, function, args=[], kwargs={})

实例化一个对象时，经过interval时间之后，才执行function函数。	

#实例

每2s执行一次out_msg函数，向终端输出一条信息。

	import threading
	count = 0
	def out_msg():
		global count
		count += 1
		print "thread delay test %d" %count

	def dump_thread(func,delay_seconds, save_thread_ptr):
		func()
		save_thread = threading.Timer(delay_seconds, dump_thread,
				args=(func,delay_seconds,save_thread_ptr))
		save_thread.start()

		if save_thread_ptr:
			save_thread_ptr.pop(0)
		save_thread_ptr.append(save_thread)

	if __name__=='__main__':
		dump_thread(out_msg, 2, [])
		print "main process continuing..."
