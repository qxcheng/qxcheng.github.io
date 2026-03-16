---
categories:
- - Python
date: 2024-04-16 16:46:52
description: 'Python进阶4-多进程'
id: '98'
tags:
- python
title: Python进阶4-多进程
---


## 多进程
Python多进程可以选择两种创建进程的方式，spawn 与 fork。**分支创建：fork**会直接复制一份自己给子进程运行，并把自己所有资源的handle 都让子进程继承，因而创建速度很快，但更占用内存资源。**分产创建：spawn**只会把必要的资源的handle 交给子进程，因此创建速度稍慢。

```Python
# Process进程
import multiprocessing
from multiprocessing import Process
import time

multiprocessing.set_start_method('spawn')  # default on WinOS or MacOS
multiprocessing.set_start_method('fork')   # default on Linux (UnixOS)

def func(message):
    time.sleep(1)
    print('PID-{}: '.format(multiprocessing.current_process().pid), message)

if __name__ == '__main__':
    for item in ['A', 'B', 'C', 'D']:
        p = Process(target=func, args=(item,))
        #p.daemon = True  # 守护进程
        p.start()
        p.join()          # 阻塞等待
        #p.terminate()    # 终止进程
# PID-1684:  A
# PID-4112:  B
# PID-15832:  C
# PID-5188:  D
```

```Python
#1 os.fork()
import os
pid = os.fork()
if pid == 0: #子进程
    print('child:%s parent:%s' % (os.getpid(), os.getppid()))
else:        #父进程
    print('parent:%s child %s' % (os.getpid(), pid))


#2 subprocess 
import subprocess
print('$ nslookup www.python.org')
r = subprocess.call(['nslookup', 'www.python.org'])
print('Exit code:', r)


print('$ nslookup')
p = subprocess.Popen(['nslookup'], stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE)
output, err = p.communicate(b'set q=mx\npython.org\nexit\n')
print(output.decode('utf-8'))
print('Exit code:', p.returncode)  

```

## 进程池

```Python
#1 Pool进程池
from multiprocessing import Pool
import multiprocessing as mp
import time

def func(message):
    time.sleep(1)
    print('PID-{}: '.format(mp.current_process().pid), message)

if __name__ == '__main__':
    p = Pool(2)

    for i in range(5):
        p.apply_async(func, args=(i,))    
        
    p.close() #调用close()之后不能添加新的Process
    p.join()  #阻塞等待  
# PID-4368:  0
# PID-15972:  1
# PID-4368:  2
# PID-15972:  3
# PID-4368:  4


#2 map
import time

def func2(args):  # multiple parameters (arguments)
    # x, y = args
    x = args[0]  # write in this way, easier to locate errors
    y = args[1]  # write in this way, easier to locate errors

    time.sleep(1)  # pretend it is a time-consuming operation
    return x - y

def run__pool():  # main process
    from multiprocessing import Pool

    cpu_worker_num = 3
    process_args = [(1, 1), (9, 9), (4, 4), (3, 3), ]

    print(f'| inputs:  {process_args}')
    start_time = time.time()
    with Pool(cpu_worker_num) as p:
        outputs = p.map(func2, process_args)
    print(f'| outputs: {outputs}    TimeUsed: {time.time() - start_time:.1f}    \n')

    '''Another way (I don't recommend)
    Using 'functions.partial'. See https://stackoverflow.com/a/25553970/9293137
    from functools import partial
    # from functools import partial
    # pool.map(partial(f, a, b), iterable)
    '''

if __name__ =='__main__':
    run__pool()


#3 ProcessPoolExecutor
from concurrent.futures import ProcessPoolExecutor, as_completed
import time 

def square(n):
    time.sleep(1)
    return n * n

if __name__ == '__main__':
    with ProcessPoolExecutor(max_workers=3) as executor:
        tasks = [executor.submit(square, num) for num in range(10)]
        for future in as_completed(tasks):
            print(future.result())
# 0 1 4 9 16 25 36 49 64 81    

```

## Pipe

顾名思义，管道Pipe 有两端，因而 main_conn, child_conn = Pipe() ，管道的两端可以放在主进程或子进程内，我在实验中没发现主管道口main_conn 和子管道口child_conn 的区别。两端可以同时放进去东西，放进去的对象都经过了深拷贝：用 conn.send()在一端放入，用 conn.recv() 另一端取出，管道的两端可以同时给多个进程。conn是 connect的缩写。

如果追求运行更快，**那么最好使用管道Pipe而非下面介绍的队列Queue**，详细请移步[Python pipes and queues performance](https://link.zhihu.com/?target=https://www.raspberrypi.org/forums/viewtopic.php?t=141576) ↓

```Python
#1 Pipe，管道是全双工
from multiprocessing import Process, Pipe

def f(conn):
    conn.send([42, None, 'hello'])
    conn.close()

parent_conn, child_conn = Pipe()
p = Process(target = f, args = (child_conn,))
p.start()
print(parent_conn.recv())
p.join()


#2
默认情况下 duplex==True，若不开启双向管道，那么传数据的方向只能 conn1 ← conn2 。conn2.poll()==True 意味着可以
马上使用 conn2.recv() 拿到传过来的数据。conn2.poll(n) 会让它等待n秒钟再进行查询。

from multiprocessing import Pipe

conn1, conn2 = Pipe(duplex=True)  # 开启双向管道，管道两端都能存取数据。默认开启

conn1.send('A')
print(conn1.poll())  # 会print出 False，因为没有东西等待conn1去接收
print(conn2.poll())  # 会print出 True ，因为conn1 send 了个 'A' 等着conn2 去接收
print(conn2.recv(), conn2.poll(2))  # 会等待2秒钟再开始查询，然后print出 'A False'

```

## Queue

可以 import queue 调用Python内置的队列，在多线程里也有队列 from multiprocessing import Queue。下面提及的都是多线程的队列。

队列Queue 的功能与前面的管道Pipe非常相似：无论主进程或子进程，都能访问到队列，放进去的对象都经过了深拷贝。不同的是：管道Pipe只有两个断开，而队列Queue 有基本的队列属性，更加灵活，详细请移步Stack Overflow [Multiprocessing - Pipe vs Queue](https://link.zhihu.com/?target=https://stackoverflow.com/questions/8463008/multiprocessing-pipe-vs-queue)。

```Python
##1 Queue 
from multiprocessing import Process, Queue
import os

def write(q):          
    for value in ['A', 'B', 'C', 'D']:
        print('Process %s put %s to queue.' % (os.getpid(), value))
        q.put(value)
        
def read(q):          
    while True:
        value = q.get(True)
        print('Process %s get %s from queue.' % (os.getpid(), value))

if __name__ == '__main__':
    q = Queue(maxsize=4)    # 父进程创建Queue，并传给各个子进程 q.qsize()
    
    pw = Process(target=write, args=(q,))
    pr = Process(target=read, args=(q,))
    pw.start()     
    pr.start()     
    
    pw.join()      # 等待pw结束
    pr.terminate() # pr进程是死循环，强行终止
    
# Process 16304 put A to queue.
# Process 16304 put B to queue.
# Process 16304 put C to queue.
# Process 14676 get A from queue.
# Process 16304 put D to queue.
# Process 14676 get B from queue.
# Process 14676 get C from queue.
# Process 14676 get D from queue.

```

## 共享内存Manager

为了在Python里面实现多进程通信，上面提及的 Pipe Queue 把需要通信的信息从内存里深拷贝了一份给其他线程使用（需要分发的线程越多，其占用的内存越多）。而共享内存会由解释器负责维护一块共享内存（而不用深拷贝），这块内存每个进程都能读取到，读写的时候遵守管理（因此不要以为用了共享内存就一定变快）。

Manager可以创建一块共享的内存区域，但是存入其中的数据需要按照特定的格式，Value可以保存数值，Array可以保存数组，如下。这里不推荐认为自己写代码能力弱的人尝试。下面这里例子来自[Python官网的Document](https://link.zhihu.com/?target=https://docs.python.org/3/library/multiprocessing.html)。
```python
from multiprocessing import Process, Lock, shared_memory

def func(args):
    shm = shared_memory.SharedMemory(name='shm')
    buf = shm.buf
    buf[:len(args)] = bytearray(args)
    
if __name__ == '__main__':
    shm = shared_memory.SharedMemory(name='shm', create=True, size=4096)
    
    p = Process(target=func, args=([1,2,3,4,5],))
    p.start()
    p.join()

    for item in shm.buf[:10]:
        print(item, end=' ')     # 1 2 3 4 5 0 0 0 0 0

    shm.close()   # 关闭共享内存
    shm.unlink()  # 释放共享内存
```


```Python
# https://docs.python.org/3/library/multiprocessing.html?highlight=multiprocessing%20array#multiprocessing.Array

from multiprocessing import Process, Lock
from multiprocessing.sharedctypes import Value, Array
from ctypes import Structure, c_double

class Point(Structure):
    _fields_ = [('x', c_double), ('y', c_double)]

def modify(n, x, s, A):
    n.value **= 2
    x.value **= 2
    s.value = s.value.upper()
    for a in A:
        a.x **= 2
        a.y **= 2

if __name__ == '__main__':
    lock = Lock()

    n = Value('i', 7)
    x = Value(c_double, 1.0/3.0, lock=False)
    s = Array('c', b'hello world', lock=lock)
    A = Array(Point, [(1.875,-6.25), (-5.75,2.0), (2.375,9.5)], lock=lock)

    p = Process(target=modify, args=(n, x, s, A))
    p.start()
    p.join()

    print(n.value)
    print(x.value)
    print(s.value)
    print([(a.x, a.y) for a in A])
```