---
categories:
- - Python
date: 2024-04-16 16:59:14
description: 'Python进阶5-多线程'
id: '100'
tags:
- python
title: Python进阶5-多线程
---


## 1.多线程

```python
import threading

lock = threading.Lock()
num = 0

def func(n):
    lock.acquire()
    global num
    num += 1
    print('{}: '.format(n), num)
    lock.release()   

t1 = threading.Thread(target=func, args=('Tom',), name='Thread1', daemon=True)
t2 = threading.Thread(target=func, args=('Bob',), name='Thread2', daemon=True)
t1.start()  #启动线程
t2.start()
t1.join()   #阻塞等待线程
t2.join()   
# Tom:  1
# Bob:  2

# t.isAlive()    #检查线程是否仍在执行。
# t.getName()    #返回线程的名称。
# t.setName()    #设置线程的名称。
# t.exit()

# threading.get_ident()           #返回当前线程id
# threading.activeCount()         #返回活动状态的线程的数量
# threading.currentThread()       #返回调用者线程控制中的线程对象数
# threading.current_thread().name #不指定时为MainThread
# threading.enumerate()           #返回当前活动的所有线程对象的列表。
```

## 2.threadlocal

threading.local 提供了一种方便的方式来创建线程本地数据，使得每个线程都可以独立地访问和修改数据，而不会影响其他线程。

```python
import threading
import time

# 利用local类，创建一个全局对象 local_obj
local_obj = threading.local()

def func():
    local_obj.var = 0
    # 如果使用局部变量，函数调用需要传参
    func_print()

def func_print():
    for k in range(100):
        time.sleep(0.01)
        # 直接使用local_obj.var，自动获取当前线程对应的属性值
        local_obj.var += 1
    print(f'线程id：{threading.get_ident()}，thread-local数据：{local_obj.var}')

# 创建3个线程，并启动
for th in range(3):
    threading.Thread(target=func,).start()

输出：
线程id：15952，thread-local数据：100
线程id：7152，thread-local数据：100
线程id：13588，thread-local数据：100
```

## 3.信号量

Semaphore 维护了一个内部的计数器，表示可用的许可证数量。线程可以通过调用 acquire() 方法来获取许可证，表示对共享资源的占用，或者通过调用 release() 方法来释放许可证，表示共享资源不再被占用。当计数器为正时，acquire() 方法将立即返回，并将计数器减 1；当计数器为零时，acquire() 方法将阻塞当前线程，直到有其他线程释放许可证为止。

```python
import time
from random import random
from threading import Thread, Semaphore

sema = Semaphore(3)

def foo(tid):
    with sema:
        print('{} acquire sema'.format(tid))
        wt = random() * 2
        time.sleep(wt)
    print('{} release sema'.format(tid))

threads = []

for i in range(5):
    t = Thread(target=foo, args=(i,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

## 4.Condition

Condition 是 Python 中 threading 模块提供的同步原语之一，用于在多线程编程中实现线程间的协调和通信。它允许线程等待某个条件的发生，然后通知其他线程条件已经满足。

Condition 实际上是 Lock 和 Event 的组合，它维护了一个内部的锁对象和一个内部的标志位，用于表示条件是否满足。线程可以通过 acquire() 方法获取锁，并调用 wait() 方法在条件不满足时释放锁并进入等待状态；其他线程可以通过 notify() 或 notify\_all() 方法来通知等待的线程条件已经满足，然后等待线程将重新获取锁并继续执行。

```python
import time
import threading

def consumer(cond):
    t = threading.current_thread()
    with cond:
        cond.wait()  # wait()方法创建了一个名为waiter的锁，并且设置锁的状态为locked。这个waiter锁用于线程间的通讯
        print('{}: Resource is available to consumer'.format(t.name))

def producer(cond):
    t = threading.current_thread()
    with cond:
        print('{}: Making resource available'.format(t.name))
        cond.notify_all()  # 释放waiter锁，唤醒消费者

condition = threading.Condition()

c1 = threading.Thread(name='c1', target=consumer, args=(condition,))
c2 = threading.Thread(name='c2', target=consumer, args=(condition,))
p = threading.Thread(name='p', target=producer, args=(condition,))

c1.start()
time.sleep(1)
c2.start()
time.sleep(1)
p.start()
```

## 5.线程通信

```python
import threading
import queue
import random
import time

def myqueue(queue):
    while not queue.empty():
        item = queue.get()
        if item is None:
            break
        print("{} removed {} from the queue".format(threading.current_thread(), item))
        queue.task_done()
        time.sleep(2)

q = queue.Queue()          #先进先出队列
#q = queue.LifoQueue()     栈
#q = queue.PriorityQueue() 优先级队列，q.put(i,1)

for i in range(5):
    q.put(i)    

threads = []
for i in range(4):
    thread = threading.Thread(target=myqueue, args=(q,))
    thread.start()
    threads.append(thread)

for thread in threads:
    thread.join()
```

## 6.线程池

```Python
from concurrent.futures import ThreadPoolExecutor, as_completed 
import time 

def square(n):
    time.sleep(1)
    return n * n

if __name__ == '__main__':
    with ThreadPoolExecutor(max_workers=3) as executor:
        tasks = [executor.submit(square, num) for num in range(10)]
        for future in as_completed(tasks):
            print(future.result())
```