---
layout: single
author_profile: true
comments: true
title: python thread module
category: Python
tags: [Python]
---
python2.7.6文档<a href="http://docs.python.org/2/library/thread.html" target="_blank">原文</a>

1. thread.start_new_thread(function, args[, kwargs])

创建一个线程，返回一个线程ID
function: 函数名
args: function的参数列表，以tuple的形式传入，假如函数func有3个参数argv[0],argv[1],argv[2]，这样创建线程：

thread.start_new_thread(func, (argv[0],argv[1],argv[2],))

记得tuple最后加一个‘多余’的逗号，即使function没有参数，也要传入()
kwargs: 可选参数（作用暂时不明白）

2. thread.interrupt_main()

子线程调用该函数来终止主线程

3. thread.exit()

调用该函数的线程会‘安静’的退出

4. thread.get_ident()

返回当前线程的TID

5. thread.stack_size([size])

返回用于创建新线程的thread stack大小

6. thread.allocate_lock()

返回一个thread.LockType类型的对象

7. thread.LockType对象的方法包括：

lock = thread.allocate_lock()

7.1 lock.acquire([waitflag])

如果没有参数,该函数无条件加锁，同一时刻只有一个线程可以获得一个锁，即互斥锁。
如果参数waitflag存在
waitflag=0, 它不需要等待就可以获得锁
waitflag非0，和没有参数作用一样。
申请锁成功，返回True，否则返回False

7.2 lock.release()

释放锁，lock必须在之前acquire()成功

7.3 lock.locked()

返回锁lock的当前状态，如果lock被一些线程申请(acquire())了，返回True，否则返回False

&nbsp;
<pre>import thread
import time

def consumer():
    global shareV
    global lock
    for i in range(1000):
        #可以去掉参数，用0或非0值测试下，真实感受一下参数含义
        lock.acquire(1) 
        #print("tid=%d\n" %thread.get_ident())
        shareV += 1
        if lock.locked():
            lock.release()
    thread.exit()

if __name__=='__main__':
    shareV = 0 #多线程共享
    lock = thread.allocate_lock()
    for i in range(5):
        tid = thread.start_new_thread(consumer,())
    time.sleep(5)
    print("before exit, n=%d" %shareV)
</pre>
