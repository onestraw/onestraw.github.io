---
layout: single
author_profile: true
comments: true
title: python threading module
category: Python
tags: [Python]
---
## 一.概述

本文先是对官方文档(python2.7.6  threading<a href="http://docs.python.org/2/library/threading.html" target="_blank">文档</a>)进行了学习理解，然后用锁和信号量实现互斥操作，发现Lock.acquire()和Lock.release()的执行效率远远高于Semaphore.acquire()和Semaphore.release()

## 二.threading 文档

1.

	threading.active_count()
	threading.activeCount()

返回当前存在的线程

2.

	threading.Condition()

返回一个新的condition变量，一个condition变量可以使一个或多个线程等待直到有线程通知它们。

3.

	threading.current_thread()
	threading.currentThread()

返回一个Thread对象

4.

	threading.enumerate()

返回一个当前存在的线程列表(list)，包括当前线程(调用该函数的线程)创建的守护线程，dummy线程

5.

	threading.Event()

返回一个Event对象

6.

	threading.Lock()

返回一个primitive lock对象，其它试图申请同一锁的线程进入block状态，直到它释放

7.

	threading.RLock()

返回一个reentrant lock对象，reentrant lock必须被申请它的线程来释放。
一旦一个线程申请过一个reentrant lock，同一线程可再次申请它而无需等待，每次申请使用之后必须释放

8.

	threading.Semaphore([value])

返回一个信号量(Semaphore)对象，一个信号量管理一个计数器，这个计数器表示release()调用次数 减去 acquire()调用次数，再加上初始值(参数value)。
如果没有参数value,value默认为1
当acquire()返回非负数时申请成功，否则阻塞在此。

9.

	threading.BoundedSemaphore([value])

返回一个bounded信号量对象，一个bounded信号量确保它的计数器不超过它的初始值（参数value）,否则抛出ValueError异常。
大多数情况下，信号量被用于“以有限的空间存放资源”。如果没给出参数，value默认是1.

10.

	class threading.Thread(group=None, target=None, name=None, args=(), kwargs={})
		__init__(group=None, target=None, name=None, args=(), kwargs={})

group = None  在这里没用，保留以后扩展
target  线程运行的函数，被run()调用，默认为None
name  线程名，一个字符串
args   target函数的参数（tuple形式传入），默认为()
kwargs   target函数的keyword参数（dict形式传入），默认为{}

## kwargs用法

{% highlight python %}
import threading

THREAD_NUM = 5

def consumer(kw1, kw2=200):
    print("--%d--%d--\n" %(kw1, kw2))
    global shareV
    global lock
    global exitFlag
    for i in range(100000):
        lock.acquire() #or lock.acquire_lock()
        shareV += 1
        lock.release()  #or lock.release_lock()
    exitFlag -= 1
    
if __name__=='__main__':
    shareV = 0 #多线程共享
    exitFlag = THREAD_NUM
    lock = threading.Lock()  #互斥锁
    
    for tno in range(THREAD_NUM):
        t = threading.Thread(target=consumer,kwargs={'kw1':tno,'kw2':123})
        #t = threading.Thread(target=consumer,args=(tno,123))
        t.start()

    while exitFlag:
        pass
    print("before exit, shareV=%d" %shareV)
{% endhighlight %}

  start() 启动线程，每个Thread对象最多start()一次，用run()进行线程控制等。如果调用多于一次，会抛出RuntimeError

  run()    表述一个线程的活动，可以在子类中重载该函数

  join([timeout])   </strong>等待直到线程终止。它阻塞调用此函数的线程，直到线程终止（异常或正常退出）或者可选参数timeout耗尽。

  name   用于”标识”一个进程的字符串，不是id，多个线程可以用相同的name

getName()
setName()

 ident   线程标识符tid，当线程未启动时，它是None。

    is_alive()
    isAlive()

    daemon   一个布尔值，标明该线程是不是daemon
    isDaemon()
    setDaemon()    daemon是守护进程，顾名思义

## 三.实现互斥

模拟互斥的作用：使用5个线程对同一变量进行+1操作，每个线程执行10w次，我们期望得到50w的结果，观察没有互斥和使用互斥操作的区别。

### 1. 线程间没有互斥操作
{% highlight python %}
import threading

THREAD_NUM = 5

def consumer():
    global shareV
    global exitFlag
    for i in range(100000):
        shareV += 1
    exitFlag -= 1

class myThread(threading.Thread):
    def __init__(self,target=None, name=None, args=(), kwargs={}):
        threading.Thread.__init__(self, target=target,name=name,args=args,kwargs=kwargs)
        self.start()
         
if __name__=='__main__':
    shareV = 0 #多线程共享
    exitFlag = THREAD_NUM
    for tno in range(THREAD_NUM):
        myThread(target=consumer)

    thread = myThread(name="http://onestraw.net", target=consumer)

    print("存在 %d 个线程" %threading.activeCount())
    for t in threading.enumerate():
        print("Thread Name:%s, Thread id:%d" %(t.getName(),t.ident))

    #等待所有线程执行结束
    while exitFlag:
        pass
    print("before exit, shareV=%d" %shareV)

{% endhighlight %}
### 2. 使用Lock

{% highlight python %}
import threading

THREAD_NUM = 5

def consumer():
    global shareV
    global lock
    global exitFlag
    for i in range(100000):
        lock.acquire() #or lock.acquire_lock()
        shareV += 1
        lock.release()  #or lock.release_lock()
    exitFlag -= 1

class myThread(threading.Thread):
    def __init__(self,target=None, name=None, args=(), kwargs={}):
        threading.Thread.__init__(self, target=target,name=name,args=args,kwargs=kwargs)
        self.start()        
    
if __name__=='__main__':
    shareV = 0 #多线程共享
    exitFlag = THREAD_NUM
    lock = threading.Lock()  #互斥锁
    
    for tno in range(THREAD_NUM):
        myThread(target=consumer)

    print("存在 %d 个线程" %threading.activeCount())
    for t in threading.enumerate():
        print("Thread Name:%s, Thread id:%d" %(t.getName(),t.ident))

    #等待所有线程执行结束
    while exitFlag:
        pass
    print("before exit, shareV=%d" %shareV)
{% endhighlight %}

### 3. 使用Semaphore

{% highlight python %}
import threading

THREAD_NUM = 5

def consumer():
    global shareV
    global semaphore
    global exitFlag
    for i in range(100000):
        semaphore.acquire()
        shareV += 1
        semaphore.release()
    exitFlag -= 1
    
class myThread(threading.Thread):
    def __init__(self,target=None, name=None, args=(), kwargs={}):
        threading.Thread.__init__(self, target=target,name=name,args=args,kwargs=kwargs)
        self.start()       
    
if __name__=='__main__':
    shareV = 0 #多线程共享
    exitFlag = THREAD_NUM
    semaphore = threading.Semaphore()  #用信号量实现互斥
    
    for tno in range(THREAD_NUM):
        myThread(target=consumer)

    print("存在 %d 个线程" %threading.activeCount())
    for t in threading.enumerate():
        print("Thread Name:%s, Thread id:%d" %(t.getName(),t.ident))

    #等待所有线程执行结束
    while exitFlag:
        pass
    print("before exit, shareV=%d" %shareV)</pre>
{% endhighlight %}
