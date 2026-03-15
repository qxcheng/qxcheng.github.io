---
categories:
- - Python
date: 2024-04-16 13:36:22
description: 'Python基础5-异常处理'
id: '80'
tags:
- python
title: Python基础5-异常处理
---


## 1.异常

```Python
import traceback

#1 捕获异常
try:
    print(int('100years'))
    return
except ValueError as e:
    print('ValueError:', e) 
    traceback.print_exc()          # 直接打印异常
    print(traceback.format_exc())  # 返回字符串
    traceback.print_exc(file=open('error.txt','a+'))  # 写入到文件
else:
    # try语句没有异常时运行，在finally语句之前执行
    # 此处的异常将不会被捕获
    print('No error!')
finally:
    # 一定会执行
    print('Finally...')

#2 捕获多个异常
try:
    file = open('file.txt', 'rb')
except EOFError as e:
    print("An EOF error occurred.")
    raise e  # 抛出异常
except IOError as e:
    print("An error occurred.")
    raise e

#3 捕获所有异常
try:
    n = 0
    assert n != 0, 'n is zero!'  # 断言  $ python -O err.py  关闭assert 
except Exception as e:
    print(e)  # n is zero!
```

## 2.自定义异常

```Python
class MyError(RuntimeError):
    def __init__(self, msg):
        self.msg = msg

    def __str__(self):
        return self.msg

try:
    raise MyError("an error")
except MyError as e:
    print(e)  # an error
```

## 3.常见异常

```python
try:
    a = []
    print(a[0])
except IndexError as e:
    print(e)  # list index out of range

try:
    a = {}
    print(a[0])
except KeyError as e:
    print(e)  # 0

try:
    a = 1 / 0
except ZeroDivisionError as e:
    print(e)  # division by zero

try:
    class A():
        pass
    a = A()
    print(a.name)
except AttributeError as e:
    print(e)  # 'A' object has no attribute 'name'

try:
    f = open('test.txt', 'r')
except OSError as e:
    print(e)  # [Errno 2] No such file or directory: 'test.txt'
```