---
categories:
- - Python
date: 2024-04-16 13:19:45
description: 'Python基础3-进阶语法'
id: '65'
tags:
- python
title: Python基础3-进阶语法
---


## 1.迭代器

*   可迭代对象(Iterable): 即可以循环遍历的对象，包含list、tuple、dict、set、str、生成器，可迭代对象定义了`__iter__`或`__getitem__`方法， iter()可以将可迭代对象变成迭代器
    
*   迭代器(Iterator): 包含生成器，不包含list、dict、str，迭代器定义了`__next__`方法返回单个元素，可被next()调用，此外还要实现 `__iter__` 方法，返回迭代器本身。
    
*   解释器需要迭代对象 x 时，会自动调用 iter(x)。内置的 iter 函数有以下作用：
    
    1.  检查对象是否实现了 `__iter__` 方法，如果实现了就调用它，获取一个迭代器。
    2.  如果没有实现 `__iter__` 方法，但是实现了 `__getitem__` 方法，Python 会创建一个迭代器，尝试按顺序（从索引 0 开始）获取元素。
    3.  如果尝试失败，Python 抛出 TypeError 异常，通常会提示“C object is not iterable”（C对象不可迭代），其中 C 是目标对象所属的类。

### 迭代器实现

```python
class Fib(object):  
    def __init__(self, num):
        self.a, self.b = 0, 1
        self.num = num

    def __iter__(self):       
        return self           

    def __next__(self):
        self.a, self.b = self.b, self.a + self.b 
        if self.a > self.num: 
            raise StopIteration()
        return self.a

fib = Fib(10)
for i in fib:
    print(i) # 1 1 2 3 5 8
```

## 2.itertools

```python
import itertools

#1 count(1)从1开始创建一个无限的迭代器
num = itertools.count(1)
for i in num:
    if i > 5:
        break
    print(i, end=' ') # 1 2 3 4 5 

#2 cycle()把传入的序列无限重复下去
alpha = itertools.cycle('ABC') 
n = 0
for i in alpha:
    if n > 6:
        break
    n += 1
    print(i, end=' ') # A B C A B C A

#3 repeat()把一个元素无限重复下去，第二个参数可以限定重复次数
alpha = itertools.repeat('A', 3)
for i in alpha:
    print(i, end=' ') # A A A 

#4 takewhile()根据条件截取出一个有限的序列
num = itertools.count(1)
part = itertools.takewhile(lambda x: x < 5, num)
print(list(part)) # [1, 2, 3, 4]

#5 chain()把一组迭代对象连接起来
for i in itertools.chain('ABC', 'DEF'):
    print(i, end=' ') # A B C D E F 

#6 groupby()把迭代器中相邻的重复元素挑出来放在一起
for key, group in itertools.groupby('ABBCCC'):
    print(key, list(group))
# A ['A']
# B ['B', 'B']
# C ['C', 'C', 'C']

#7 groupby()的挑选规则通过函数完成,只要作用于函数的两个元素的返回值相等，就被认为是一组，而函数返回值作为组的key
for key, group in itertools.groupby('AaBBbcCcC', lambda c: c.upper()):
    print(key, list(group))
# A ['A', 'a']
# B ['B', 'B', 'b']
# C ['c', 'C', 'c', 'C']
```

## 3.生成器

生成器（Generators）是一种用于生成迭代器的函数。与常规函数不同，生成器函数在运行时不会一次性返回所有结果，而是使用 yield 语句逐个生成结果，并在需要时暂停执行状态。这样可以节省内存空间，并且在处理大量数据时效率更高。

```python
#1 使用推导式构建生成器
g = (x * x for x in range(5))

for i in g:
    print(i) # 0 1 4 9 16

#2 使用yield关键字构建生成器
def g():
    for i in range(5):
        yield i * i

for i in g():
    print(i) # 0 1 4 9 16
```

## 4.闭包

闭包是指在一个函数中定义的函数，并且内部函数可以访问外部函数作用域中的变量。换句话说，闭包是由函数和与其相关的引用环境组合而成的实体。

```python
#1 闭包
def lazy_sum(*args):
    def sum():
        _sum = 0
        for n in args:
            _sum += n
        return _sum
    return sum

f = lazy_sum(1,2,3,4,5)
print(f()) # 15

#2 循环变量的当前值未被捕获：
def count():
    fs = []
    for i in range(1, 4):
        def f():
            return i*i
        fs.append(f) 
    return fs

f1, f2, f3 = count() 
print(f1(), f2(), f3()) # 9 9 9

#3 循环变量的当前值被捕获：
def count():
    fs = []
    for i in range(1, 4):
        def f(_i=i):       
            return _i*_i
        fs.append(f)    
    return fs

f1, f2, f3 = count()    
print(f1(), f2(), f3()) # 1 4 9

#4 利用闭包实现计数器，每次调用返回递增整数：
def createCounter():
    fs = [0]
    def counter():
        fs[0] = fs[0] + 1
        return fs[0]
    return counter

f = createCounter()
for i in range(5):
    print(f()) # 1 2 3 4 5
```

## 5.装饰器

装饰器是一种Python语法糖，它允许您在不修改原始函数定义的情况下，通过在函数之前添加 @decorator\_name（装饰器函数名）来修改或增强函数的行为。装饰器本质上是一个函数，它接受一个函数作为参数，并返回一个新的函数，通常用于对原始函数进行包装、扩展或修改。

### 原理

装饰器在导入模块时立即执行，而被装饰的函数只在明确调用时运行。

```python
@decorate
def target():
    print('running target()')
等价于：
def target():
    print('running target()')
target = decorate(target)      # target赋值为了装饰器函数的返回值
```

### 使用

```python
import functools
import time

#1 装饰器 
def log(func):
    @functools.wraps(func)
    def wrapper(*args, **kw):
        print(func.__name__) 
        return func(*args, **kw)
    return wrapper

@log
def func():
    print('Hello')

func()
# func
# Hello

#2 传入参数的装饰器
def log(text):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kw):
            print(text, func.__name__)
            return func(*args, **kw)
        return wrapper
    return decorator

@log('execute')
def func():
    print('Hello')

func()
# execute func
# Hello

#3 打印函数的执行时间
def metric(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kw):
        start = time.time()
        f = fn(*args, **kw)  
        print(time.time() - start)
        return f
    return wrapper

@metric
def func():
    sum = 0
    for i in range(1000000):
        sum += i*i*i*i
        sum -= i*i*i*i
    print('sum: ', sum)

func()
# sum:  0
# 0.25935

#4 装饰器类
class logit(object): 
    def __init__(self, msg):
        self.msg = msg
        print('init:', self.msg)

    def __call__(self, func):
        def wrapper(*args, **kw):       
            print(func.__name__) 
            print(self.msg)
            return func(*args, **kw)
        return wrapper

@logit('log')
def func():
    print("Hello")

func()
# init: log
# func
# log
# Hello
```

### lru\_cache

lru\_cache 装饰器可以应用于任何函数，并自动缓存函数的结果，以避免重复计算。当相同的参数传递给被装饰的函数时，如果缓存中已经存在相应的结果，则直接返回缓存中的值，而不执行实际的函数调用。如果缓存中没有对应的结果，则调用函数来计算结果，并将结果存储在缓存中以备后用。

```python
from functools import lru_cache

@lru_cache(maxsize=3)  # 最多缓存3个结果
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(10))  # 第10个斐波那契数，结果将被缓存
print(fibonacci(5))   # 第5个斐波那契数，结果将被缓存
print(fibonacci(10))  # 缓存命中，直接返回结果，不执行函数调用
```

### singledispatch

singledispatch 装饰器的作用是为函数注册多个专门的实现版本，每个版本对应不同的参数类型。当调用被装饰的函数时，singledispatch 会根据第一个参数的类型来选择合适的实现版本来执行。如果没有匹配的专门实现，它会调用默认的通用实现。

```python
from functools import singledispatch

@singledispatch
def my_func(arg):
    print("Default:", arg)

@my_func.register(int)
def _(arg):
    print("Integer:", arg)

@my_func.register(str)
def _(arg):
    print("String:", arg)

@my_func.register(list)
def _(arg):
    print("List:", arg)

# 调用函数
my_func(10)          # 输出 "Integer: 10"
my_func("hello")     # 输出 "String: hello"
my_func([1, 2, 3])   # 输出 "List: [1, 2, 3]"
my_func(3.14)        # 输出 "Default: 3.14"
```

## 6.上下文管理器

上下文管理器（Context Managers）是一种用于管理资源的 Python 编程模式。它可以确保在使用资源（例如文件、网络连接、数据库连接等）的过程中，资源被正确地分配、使用和释放，即使在遇到异常或其他意外情况时也能保证资源的正确释放。

```python
#1 基于类，定义enter和exit方法
class File(object):
    def __init__(self, name, method):
        self.file = open(name, method)

    def __enter__(self):
        return self.file

    #exit返回True清空处理异常，否则抛出异常
    def __exit__(self, type, val, trace): 
        self.file.close()
        return True

with File('test.py', 'r') as f:
    f.undefined_function()
    print('Hello')  # 这里不执行了

#2 基于生成器
from contextlib import contextmanager

@contextmanager
def open_file(name):
    f = open(name, 'w')
    yield f
    f.close()

with open_file('test.txt') as f:
    f.write('hello!')

#3 首尾代码固定时：
@contextmanager
def tag(name):
    print("<%s>" % name)
    yield
    print("</%s>" % name)

with tag("h1"):
    print("hello world")
# <h1>
# hello world
# </h1>
```