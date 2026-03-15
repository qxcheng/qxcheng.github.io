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