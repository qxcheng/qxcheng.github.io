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


## 1.多进程

Python多进程可以选择两种创建进程的方式，spawn 与 fork。**分支创建：fork**会直接复制一份自己给子进程运行，并把自己所有资源的handle 都让子进程继承，因而创建速度很快，但更占用内存资源。**分产创建：spawn**只会把必要的资源的handle 交给子进程，因此创建速度稍慢。

```Python
#1 Process进程
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

#2 os.fork()
import os
pid = os.fork()
if pid == 0: #子进程
    print('child:%s parent:%s' % (os.getpid(), os.getppid()))
else:        #父进程
    print('parent:%s child %s' % (os.getpid(), pid))
```

## 2.进程池

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

#2 ProcessPoolExecutor
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

## 3.Pipe

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
```

## 4.Queue

```Python
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

## 5.共享内存Manager

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