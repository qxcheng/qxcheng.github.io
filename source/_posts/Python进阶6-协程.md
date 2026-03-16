---
categories:
- - Python
date: 2024-04-17 10:45:39
description: 'Python进阶6-协程'
id: '104'
tags:
- python
title: Python进阶6-协程
---


## 1.协程使用

```python
import asyncio

async def a():
    print('Coro a is running')
    await asyncio.sleep(2)  # await 表示调用协程
    print('Coro a is done')
    return 'A'

async def b():
    print('Coro b is running')
    await asyncio.sleep(1)
    print('Coro b is done')
    return 'B'

if __name__ == '__main__':
    asyncio.run(main())

    # 老写法
    '''
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
    loop.close()
    '''
```

```python
async def main():
    # 用来并发运行任务 可以收集结果 屏蔽掉了异常
    return_value_a, return_value_b = await asyncio.gather(a(), b(), return_exceptions=True)  
    print(return_value_a, return_value_b)
```

```python
async def main():
    task1 = asyncio.create_task(a())
    task2 = asyncio.create_task(b())

    done, pending = await asyncio.wait([task1, task2], return_when=asyncio.tasks.ALL_COMPLETED)  
    # FIRST_COMPLETED 第一个协程完成就返回
    # ALL_COMPLETED 等待全部任务完成返回（默认情况）
    # FIRST_EXCEPTION 出现第一个异常就返回

    for task in done:
        print(f'Task {task.get_name()} result: {task.result()}')
```

```python
async def main():
    # 高阶API 实际使用了loop.create_task
    task1 = asyncio.create_task(a())
    task2 = asyncio.create_task(b())
    await task1
    await task2
```

```python
async def main():
    task1 = asyncio.ensure_future(a())  # 参数是协程（调用loop.create_task）、Future 对象（直接返回）、awaitable 对象（会 await 这个对象的__await__方法，再执行一次 `ensure_future`，最后返回 Task 或者 Future）
    task2 = asyncio.ensure_future(b())
    await task1
    await task2
```

```python
async def main():
    loop = asyncio.get_event_loop()
    task1 = loop.create_task(a())    # 参数是协程
    task2 = loop.create_task(b()) 
    task2.cancel()  # 取消任务
    await task1
    #await task2
```

```python
async def main():
    loop = asyncio.get_event_loop()
    task1 = loop.create_task(a())  
    task2 = asyncio.shield(b())  # 保护任务能顺利完成，但依然返回异常结果
    task2.cancel()  # 取消任务 仍然执行
    await task1
    #await task2
```

## 2.协程原理

```Python
#1 整个流程无锁，由一个线程执行，producer和consumer协作完成任务，而非线程的抢占式多任务
'''
1.producer通过c.send(None)启动生成器
2.producer通过c.send(n)切换到consumer，
3.consumer通过yield拿到send消息，再通过yield切换到producer并把结果返回
4.producer在c.send(n)返回时拿到consumer结果，继续生产下一条消息
5.producer通过c.close()关闭consumer，结束
'''

import time

def consumer():    #生成器
    r = ''
    while True:
        n = yield r
        if not n:
            return
        print('[CONSUMER] Consuming %s...' % n)
        r = 'OK'

def produce(c):    #接收一个生成器
    c.send(None)   # 激活协程 等价于 next(c)
    n = 0
    while n < 5:
        n = n + 1
        print('[PRODUCER] Producing %s...' % n)
        r = c.send(n)
        print('[PRODUCER] Consumer return: %s' % r)
    c.close()

c = consumer()
produce(c)

#2 实例：Python实现的grep

def grep(pattern):
    while True:
        line = (yield)
        if pattern in line:
            print(line)

search = grep('coroutine')
next(search)                    # 启动协程
search.send("I love you")       # 传入数据
search.send("I love coroutine instead!")
#output: I love coroutine instead!
search.close()                  # 关闭协程
```

### task
```python
import asyncio


class Task(asyncio.futures.Future):
    def __init__(self, gen, *,loop):
        super().__init__(loop=loop)
        self._gen = gen
        self._loop.call_soon(self._step)

    def _step(self, val=None, exc=None):
        try:
            if exc:
                f = self._gen.throw(exc)
            else:
                f = self._gen.send(val)
        except StopIteration as e:
            self.set_result(e.value)
        except Exception as e:
            self.set_exception(e)
        else:
            f.add_done_callback(
                 self._wakeup)

    def _wakeup(self, fut):
        try:
            res = fut.result()
        except Exception as e:
            self._step(None, e)
        else:
            self._step(res, None)

async def foo():
    await asyncio.sleep(2)
    print('Hello Foo')


async def bar():
    await asyncio.sleep(1)
    print('Hello Bar')


loop = asyncio.get_event_loop()
tasks = [Task(foo(), loop=loop),
         loop.create_task(bar())]
loop.run_until_complete(
        asyncio.wait(tasks))
loop.close()

```

### loop
```python
import asyncio
from collections import deque


def done_callback(fut):
    fut._loop.stop()


class Loop:
    def __init__(self):
        self._ready = deque()
        self._stopping = False

    def create_task(self, coro):
        Task = asyncio.tasks.Task
        task = Task(coro, loop=self)
        return task

    def run_until_complete(self, fut):
        tasks = asyncio.tasks
        # 获取任务
        fut = tasks.ensure_future(
                    fut, loop=self)
        # 增加任务到self._ready
        fut.add_done_callback(done_callback)
        # 跑全部任务
        self.run_forever()
        # 从self._ready中移除
        fut.remove_done_callback(done_callback)

    def run_forever(self):
        try:
            while 1:
                self._run_once()
                if self._stopping:
                    break
        finally:
            self._stopping = False

    def call_soon(self, cb, *args):
        self._ready.append((cb, args))

    def _run_once(self):
        ntodo = len(self._ready)
        for i in range(ntodo):
            t, a = self._ready.popleft()
            t(*a)

    def stop(self):
        self._stopping = True

    def close(self):
        self._ready.clear()

    def call_exception_handler(self, c):
        pass

    def get_debug(self):
        return False

async def foo():
    print('Hello Foo')

async def bar():
    print('Hello Bar')

loop = Loop()
tasks = [loop.create_task(foo()),
         loop.create_task(bar())]
loop.run_until_complete(
        asyncio.wait(tasks))
loop.close()

```

### yield from 
yield from x 表达式对 x 对象所做的第一件事是，调用 iter(x)，从中获取迭代器。

```python
>>> def gen():
... yield from 'AB'
... yield from range(1, 3)
...
>>> list(gen())
['A', 'B', 1, 2]

```


## 3.asynccontextmanager

`asynccontextmanager` 是 Python 中 `contextlib` 模块提供的一个装饰器，用于创建异步上下文管理器。异步上下文管理器类似于同步上下文管理器，但可以在异步代码中使用。

```python
from contextlib import asynccontextmanager
import asyncio

# 定义异步上下文管理器
@asynccontextmanager
async def async_manager():
    print("Entering async context")
    try:
        # 异步操作：获取资源
        await asyncio.sleep(1)
        yield "resource"
    finally:
        # 异步操作：释放资源
        await asyncio.sleep(1)
        print("Exiting async context")

# 使用异步上下文管理器
async def main():
    async with async_manager() as resource:
        print("Inside async context:", resource)

# 运行主函数
asyncio.run(main())

$
Entering async context
Inside async context: resource
Exiting async context
```

## 4.异步装饰器

```python
# 异步装饰器
import asyncio  

def async_decorator(func):  
    async def wrapper(*args, **kwargs):  
        # 在这里添加异步操作的代码  
        print("Starting asynchronous operation...")  
        result = await func(*args, **kwargs)  
        print("Asynchronous operation completed: ", result)  
        return result  
    return wrapper

@async_decorator  
async def my_function():  
    await asyncio.sleep(1)  
    print("my_function completed.")  
    return "Hello, world!"  

asyncio.run(my_function())

$
Starting asynchronous operation...
my_function completed.
Asynchronous operation completed:  Hello, world!
```

```python
# 带参数异步装饰器
import asyncio  

def async_decorator_with_args(arg1, arg2):  
    def inner_decorator(func):  
        async def wrapper(*args, **kwargs):  
            print(f"Starting asynchronous operation with {arg1} and {arg2}...")  
            result = await func(*args, **kwargs)  
            print("Asynchronous operation completed: ", result)  
            return result  
        return wrapper  
    return inner_decorator

@async_decorator_with_args("Parameter 1", "Parameter 2")  
async def my_function():  
    await asyncio.sleep(1)  
    print("my_function completed.")  
    return "Hello, world!"

asyncio.run(my_function())

$
Starting asynchronous operation with Parameter 1 and Parameter 2...
my_function completed.
Asynchronous operation completed:  Hello, world!
```

## 5.执行回调函数

```python
import asyncio
from functools import partial

def callback(future, n):
    print(f'Result: {future.result()}, {n}')

async def a():
    await asyncio.sleep(1)
    return 'A'

async def main():
    task = asyncio.create_task(a())
    task.add_done_callback(partial(callback, n=1))
    await task

asyncio.run(main())

$
Result: A, 1
```

## 6.执行同步代码

```python
import asyncio
import time

def a():
    time.sleep(1)
    return 'A'

async def b():
    await asyncio.sleep(1)
    return 'B'

def show_perf(func):
    print('*' * 20)
    start = time.perf_counter()
    asyncio.run(func())
    print(f'{func.__name__} Cost: {time.perf_counter() - start}')

async def c1():
    loop = asyncio.get_running_loop()
    await asyncio.gather(
        loop.run_in_executor(None, a),
        b()
    )

show_perf(c1)

********************
c1 Cost: 1.0054080998525023
```

## 锁

```python
import asyncio
import functools

def unlock(lock):
    print('callback releasing lock')
    lock.release()

async def test(locker, lock):
    print('{} waiting for the lock'.format(locker))
    with await lock:
        print('{} acquired lock'.format(locker))
    print('{} released lock'.format(locker))

async def main(loop):
    lock = asyncio.Lock()
    await lock.acquire()
    loop.call_later(0.1, functools.partial(unlock, lock))
    await asyncio.wait([test('l1', lock), test('l2', lock)])

loop = asyncio.get_event_loop()
loop.run_until_complete(main(loop))
loop.close()
```

## 条件

```python
import asyncio
import functools

async def consumer(cond, name, second):
    await asyncio.sleep(second)
    with await cond:
        await cond.wait()
        print('{}: Resource is available to consumer'.format(name))

async def producer(cond):
    await asyncio.sleep(2)
    for n in range(1, 3):
        with await cond:
            print('notifying consumer {}'.format(n))
            cond.notify(n=n)
        await asyncio.sleep(0.1)

async def producer2(cond):
    await asyncio.sleep(2)
    with await cond:
        print('Making resource available')
        cond.notify_all()

async def main(loop):
    condition = asyncio.Condition()

    task = loop.create_task(producer(condition))
    consumers = [consumer(condition, name, index)
                 for index, name in enumerate(('c1', 'c2'))]
    await asyncio.wait(consumers)
    task.cancel()

    task = loop.create_task(producer2(condition))
    consumers = [consumer(condition, name, index)
                 for index, name in enumerate(('c1', 'c2'))]
    await asyncio.wait(consumers)
    task.cancel()

loop = asyncio.get_event_loop()
loop.run_until_complete(main(loop))
loop.close()
```
## event

```python
import asyncio
import functools

def set_event(event):
    print('setting event in callback')
    event.set()

async def test(name, event):
    print('{} waiting for event'.format(name))
    await event.wait()
    print('{} triggered'.format(name))

async def main(loop):
    event = asyncio.Event()
    print('event start state: {}'.format(event.is_set()))
    loop.call_later(
        0.1, functools.partial(set_event, event)
    )
    await asyncio.wait([test('e1', event), test('e2', event)])
    print('event end state: {}'.format(event.is_set()))

loop = asyncio.get_event_loop()
loop.run_until_complete(main(loop))
loop.close()
```

## queue

```python
import asyncio
import random
import aiohttp

NUMBERS = random.sample(range(100), 7)
URL = 'http://httpbin.org/get?a={}'
sema = asyncio.Semaphore(3)

async def fetch_async(a):
    async with aiohttp.request('GET', URL.format(a)) as r:
        data = await r.json()
    return data['args']['a']

async def collect_result(a):
    with (await sema):
        return await fetch_async(a)

async def produce(queue):
    for num in NUMBERS:
        print('producing {}'.format(num))
        item = (num, num)
        await queue.put(item)

async def consume(queue):
    while 1:
        item = await queue.get()
        num = item[0]
        rs = await collect_result(num)
        print('consuming {}...'.format(rs))
        queue.task_done()

async def run():
    queue = asyncio.PriorityQueue()
    consumer = asyncio.ensure_future(consume(queue))
    await produce(queue)
    await queue.join()
    consumer.cancel()

loop = asyncio.get_event_loop()
loop.run_until_complete(run())
loop.close()
```

## 7.多进程+协程

```python
import time
import asyncio
import aiohttp  
from multiprocessing import Pool

all_urls = ['https://www.baidu.com'] * 400

async def get_html(url, sem):
    async with(sem):    
        async with aiohttp.ClientSession() as session:  
            async with session.get(url) as resp:  
                html = await resp.text()             

def main(urls):
    loop = asyncio.get_event_loop()                # 获取事件循环
    sem = asyncio.Semaphore(10)                    # 控制并发的数量
    tasks = [get_html(url, sem) for url in urls]   # 把所有任务放到一个列表中
    loop.run_until_complete(asyncio.wait(tasks))   # 激活协程
    loop.close()                                   # 关闭事件循环

if __name__ == '__main__':
    start = time.time()
    p = Pool(4)
    for i in range(4):
        p.apply_async(main, args=(all_urls[i*100:(i+1)*100],))     
    p.close() 
    p.join()  
    print(time.time()-start)   # 2.87s
```

## 多线程+协程

```python
import time
import requests
from concurrent.futures import ThreadPoolExecutor
import asyncio

NUMBERS = range(12)
URL = 'http://httpbin.org/get?a={}'

def fetch(a):
    r = requests.get(URL.format(a))
    return r.json()['args']['a']


async def run_scraper_tasks(executor):
    loop = asyncio.get_event_loop()

    blocking_tasks = []
    for num in NUMBERS:
        task = loop.run_in_executor(executor, fetch, num)
        task.__num = num
        blocking_tasks.append(task)

    completed, pending = await asyncio.wait(blocking_tasks)
    results = {t.__num: t.result() for t in completed}
    for num, result in sorted(results.items(), key=lambda x: x[0]):
        print('fetch({}) = {}'.format(num, result))

start = time.time()
executor = ThreadPoolExecutor(3)
event_loop = asyncio.get_event_loop()

event_loop.run_until_complete(
    run_scraper_tasks(executor)
)

print('Use asyncio+requests+ThreadPoolExecutor cost: {}'.format(time.time() - start))
```