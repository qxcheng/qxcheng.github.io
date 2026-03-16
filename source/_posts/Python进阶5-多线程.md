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


## 多线程
 
RLock（可重入锁）
acquire () 能够不被阻塞的被同一个线程调用多次。但是要注意的是 release () 需要调用与 acquire () 相同的次数才能释放锁。

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

## threadlocal
local_school = threading.local() 这是一个全局对象(无需global声明)，多线程读写互不干扰，可随意添加属性（子线程中访问不到主线程中的属性）
local_school.name = 'abc'

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

## LocalProxy

使用`threading.local()`对象虽然可以基于线程存储全局变量，但是在Web应用中可能会存在如下问题：

1. 有些应用使用的是greenlet协程，这种情况下无法保证协程之间数据的隔离，因为不同的协程可以在同一个线程当中。
2. 即使使用的是线程，WSGI应用也无法保证每个http请求使用的都是不同的线程，因为后一个http请求可能使用的是之前的http请求的线程，这样的话存储于`thread local`中的数据可能是之前残留的数据。

==使用==

```python
from werkzeug.local import Local, LocalManager
 
local = Local()
# 初始化
local_manager = LocalManager([local])
# 后续添加local对象：local_manager.locals.append()
# 当LocalManager对象清理的时候会将所有存储于locals中的当前context的数据都清理掉

def application(environ, start_response):
    # 当local.request被赋值之后，其可以在当前context中作为全局数据使用，即同一个greenlet(如果有)中
    local.request = request = Request(environ)

	#
	release_local(local)  # 释放当前context的local数据 release_local实际调用local对象的__release_local__ method

    
# make_middleware会确保当request结束时，所有存储于local中的对象的reference被清除
application = local_manager.make_middleware(application)
```

==Local的实现==

```python
# 在有greenlet的情况下，get_indent实际获取的是greenlet的id，而没有greenlet的情况下获取的是thread id
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident

class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __iter__(self):
        return iter(self.__storage__.items())

    # 当调用Local对象时，返回对应的LocalProxy
    def __call__(self, proxy):
        """Create a proxy for a name."""
        return LocalProxy(self, proxy)

    # Local类中特有的method，用于清空greenlet id或线程id对应的dict数据
    def __release_local__(self):
        self.__storage__.pop(self.__ident_func__(), None)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}

    def __delattr__(self, name):
        try:
            del self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)
            

```

==LocalStack==
LocalStack与Local对象类似，都是可以基于Greenlet协程或者线程进行全局存储的存储空间(实际LocalStack是对Local进行了二次封装），区别在于其数据结构是栈的形式。

```python
>>> ls = LocalStack()
>>> ls.push(42)
>>> ls.top
42
>>> ls.push(23)
>>> ls.top
23
>>> ls.pop()
23
>>> ls.top
42

```

==LocalProxy==
LocalProxy用于代理Local对象和LocalStack对象，而所谓代理就是作为中间的代理人来处理所有针对被代理对象的操作，

初始化：通过Local或者LocalStack对象的`__call__` method
```python
from werkzeug.local import Local
l = Local()

# these are proxies
request = l('request')
user = l('user')


from werkzeug.local import LocalStack
_response_local = LocalStack()

# this is a proxy
response = _response_local()
```

初始化：通过LocalProxy类进行初始化
```python
l = Local()
request = LocalProxy(l, 'request')

```

初始化：使用callable对象作为参数
通过传递一个函数，我们可以自定义如何返回Local或LocalStack对象
```python
request = LocalProxy(get_current_request())

```

代码解析：
```python
# LocalProxy部分代码

@implements_bool
class LocalProxy(object):
    __slots__ = ('__local', '__dict__', '__name__', '__wrapped__')

    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)
        if callable(local) and not hasattr(local, '__release_local__'):
            # "local" is a callable that is not an instance of Local or
            # LocalManager: mark it as a wrapped function.
            object.__setattr__(self, '__wrapped__', local)

    def _get_current_object(self):
        """Return the current object.  This is useful if you want the real
        object behind the proxy at a time for performance reasons or because
        you want to pass the object into a different context.
        """
        # 由于所有Local或LocalStack对象都有__release_local__ method, 所以如果没有该属性就表明self.__local为callable对象
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            # 此处self.__local为Local或LocalStack对象
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    @property
    def __dict__(self):
        try:
            return self._get_current_object().__dict__
        except RuntimeError:
            raise AttributeError('__dict__')

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    def __delitem__(self, key):
        del self._get_current_object()[key]

    if PY2:
        __getslice__ = lambda x, i, j: x._get_current_object()[i:j]

        def __setslice__(self, i, j, seq):
            self._get_current_object()[i:j] = seq

        def __delslice__(self, i, j):
            del self._get_current_object()[i:j]

    # 截取部分操作符代码
    __setattr__ = lambda x, n, v: setattr(x._get_current_object(), n, v)
    __delattr__ = lambda x, n: delattr(x._get_current_object(), n)
    __str__ = lambda x: str(x._get_current_object())
    __lt__ = lambda x, o: x._get_current_object() < o
    __le__ = lambda x, o: x._get_current_object() <= o
    __eq__ = lambda x, o: x._get_current_object() == o

```

动态更新：

```python
# use Local object directly
from werkzeug.local import LocalStack
user_stack = LocalStack()
user_stack.push({'name': 'Bob'})
user_stack.push({'name': 'John'})

def get_user():
    # do something to get User object and return it
    return user_stack.pop()


# 直接调用函数获取user对象
user = get_user()
print(user['name'])  # John
print(user['name'])  # John


```python
from werkzeug.local import LocalStack, LocalProxy
user_stack = LocalStack()
user_stack.push({'name': 'Bob'})
user_stack.push({'name': 'John'})

def get_user():
    # do something to get User object and return it
    return user_stack.pop()

# 通过LocalProxy使用user对象
user = LocalProxy(get_user)
print(user['name'])  # John
print(user['name'])  # Bob

```

## 信号量

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

## Condition

一个线程等待特定条件，而另一个线程发出特定条件满足的信号。最好说明的例子就是「生产者 / 消费者」模型：

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


## event
一个线程发送 / 传递事件，另外的线程等待事件的触发。

```python
import time
import threading
from random import randint


TIMEOUT = 2

def consumer(event, l):
    t = threading.current_thread()
    while 1:
        event_is_set = event.wait(TIMEOUT)
        if event_is_set:
            try:
                integer = l.pop()
                print('{} popped from list by {}'.format(integer, t.name))
                event.clear()  # 重置事件状态
            except IndexError:  # 为了让刚启动时容错
                pass


def producer(event, l):
    t = threading.current_thread()
    while 1:
        integer = randint(10, 100)
        l.append(integer)
        print('{} appended to list by {}'.format(integer, t.name))
        event.set()  # 设置事件
        time.sleep(1)


event = threading.Event()
l = []

threads = []

for name in ('consumer1', 'consumer2'):
    t = threading.Thread(name=name, target=consumer, args=(event, l))
    t.start()
    threads.append(t)

p = threading.Thread(name='producer1', target=producer, args=(event, l))
p.start()
threads.append(p)

for t in threads:
    t.join()

```

## 线程通信

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
## 线程池

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

##   
from concurrent.futures import ThreadPoolExecutor, wait, ALL_COMPLETED
pool = ThreadPoolExecutor(max_workers=12)
tasks = []
for k, v in media_list.items():
    osspath = f"anki_source/{user_id}/apkg_{pkg_id}/{v}"
    tasks.append(pool.submit(upload_file, osspath))
wait(tasks, return_when=ALL_COMPLETED)
pool.shutdown()          
```

Eventloop 可以说是 asyncio 应用的核心，Eventloop 实例提供了注册、取消和执行任务和回调的方法。

协程 (Coroutine) 本质上是一个函数，特点是在代码块中可以将执行权交给其他协程

异步操作结束后会把最终结果设置到这个 Future 对象上

开发者并不需要直接操作 Future 这种底层对象，而是用 Future 的子类 Task 协同的调度协程以实现并发。