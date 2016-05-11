---
layout: single
author_profile: true
comments: true
title: python threading 解决生产者消费者问题
category: Python
tags: [Python]
---
生产者消费者问题（Producer-consumer problem),，也称有限缓冲问题(Bounded-buffer problem)，是一个多线程同步问题的经典案例。

使用pyhon 的threading模块描述该问题，将线程函数变为从threading.Thread继承的子类，在run()方法中实现线程的功能。
{% highlight python %}
import threading
import time
#可用缓冲区大小
BufferSize = 10
#缓冲区中资源个数
itemNum = 0

class myThread(threading.Thread):
    def __init__(self,tno):
        threading.Thread.__init__(self)
        self.tno = tno
        self.stopFlag = 0
        self.start()
        
    def stop(self):
        self.stopFlag = 1    

class producer(myThread):
    def __init__(self,tno):
        myThread.__init__(self,tno)
    def run(self):
        global putSemaphore
        global takeSemaphore
        global mutex
        global itemNum
        while not self.stopFlag:
            putSemaphore.acquire()
            mutex.acquire()
            itemNum += 1
            print("producer %d put 1 item to buffer. item num=%d\n" %(self.tno,itemNum))
            mutex.release()
            takeSemaphore.release()

class consumer(myThread):
    def __init__(self,tno):
        myThread.__init__(self,tno)
    def run(self):
        global putSemaphore
        global takeSemaphore
        global mutex
        global itemNum
        while not self.stopFlag:
            takeSemaphore.acquire()
            mutex.acquire()
            itemNum -= 1
            print("consumer %d take 1 item from buffer. item num=%d\n" %(self.tno,itemNum))
            mutex.release()
            putSemaphore.release()

if __name__=='__main__':
    #可向缓冲区放置的个数
    putSemaphore = threading.Semaphore(BufferSize)
    #可从缓冲区取走的个数
    takeSemaphore = threading.Semaphore(itemNum)
    #互斥锁
    mutex = threading.Lock()

    thread=[producer,consumer]
    tlist=[]
    for i in range(10):
        t = thread[i%2](i+1)
        tlist.append(t)

    time.sleep(3)
    for t in tlist:
        t.stop()

{% endhighlight %}

**主要收获**有：

1. run()和start()的关系，启动线程需要显式调用start()，当线程被执行时它调用run()方法。为灵活控制线程，一般在 threading.Thread的子类中重载run()方法。如果没有重载run()，threading.Thread的run()方法自动调用 target函数，但是如果重载了run()，那么run()不再自动执行target函数。

2. threading.Thread 没有提供中止退出线程的方法，通常用下面模型中止线程执行：

{% highlight python %}
import threading
import time
class ThreadFunction(threading.Thread):
    def __init__(self):
        threading.Thread.__init__(self)
        self.stopFlag = 0
        self.num =0
        self.start()
    def run(self):
        while not self.stopFlag:
            self.num += 1
            print("num=%d\n" %self.num)
    def stop(self):
        self.stopFlag = 1
    
if __name__=='__main__':
    t = ThreadFunction()
    time.sleep(1)
    t.stop()
{% endhighlight %}
