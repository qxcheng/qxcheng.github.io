---
categories:
- - Python
date: 2024-04-16 11:42:29
description: 'Python基础2-基础语法'
id: '62'
tags:
- python
title: Python基础2-基础语法
---


## 1.条件语句

```python
#1 if elif else
score = 55
if score > 80:
    print("优秀")
elif score > 60:
    print("及格")
else:
    print("未及格")

#2 条件表达式
result = "及格" if score>60 else "未及格"
print(result) # 未及格
```

## 2.循环语句

```python
# for循环
for i in range(5):
    print(i, end=' ') # 0 1 2 3 4 
print('\n')

# while循环
i = 0
while i < 5:
    print(i, end=' ') # 0 1 2 3 4
    i += 1
print('\n')

# break语句
for i in range(5):
    if i > 2:
        break
    print(i, end=' ') # 0 1 2
print('\n')

# continue语句
for i in range(5):
    if i % 2 == 0:
        continue
    print(i, end=' ') # 1 3 
print('\n')

# for/else语句
for i in range(5):
    if i > 10:
        break
    print(i, end=' ') # 0 1 2 3 4
else:
    # else块在循环正常结束时执行
    print("我执行了")
```

## 3.函数

```python
#1 函数定义与调用
def func(name, age=20, *args, **kw):
    print(name, age)
    print(args)
    print(kw)
func('Mary', 21, 1,2, a='A',b='B')
# Mary 21
# (1, 2)
# {'a': 'A', 'b': 'B'}

#2 lambda匿名函数
add = lambda x, y: x + y   
print(add(2,3)) # 5

# 在定义时绑定值
y = 10
add = lambda x, y=y: x + y 
print(add(2))   # 12

funcs = [lambda x: x+n for n in range(5)]
for func in funcs:
    print(func(5)) # 9 9 9 9 9

# 在定义时绑定值    
funcs = [lambda x, n=n: x+n for n in range(5)]
for func in funcs:
    print(func(5)) # 5 6 7 8 9

#3 偏函数
from functools import partial

int2 = partial(int, base=2)
print(int2('110'))       # 6

max10 = partial(max, 10) # 把10加入比较
print(max10(1,2,3))      # 10
```

## 4.命令行参数

```python
import sys
print(sys.argv[0])
print(sys.argv[1])
```

```python
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("-u", "--url", type=str, default="xxx.com", help="input a url")
args = parser.parse_args()
print(args.url)

$ python test.py -u yyy.com
$ yyy.com
```