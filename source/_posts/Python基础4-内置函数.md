---
categories:
- - Python
date: 2024-04-16 13:31:39
description: 'Python基础4-内置函数'
id: '78'
tags:
- python
title: Python基础4-内置函数
---


## 1.输入输出

```Python
# 输入  
name = input("请输入您的姓名：") # 返回字符串

# 输出
print("Hello: ", name)          

# 格式化输出
print('%2d-%02d' % (1, 1))      # ' 1-01'
print('A', 1, sep=',', end='!') # A,1!
print(f'hello:{name}')

# format格式化输出
print("{0},{1}".format('A','B'))     # A,B
print("{a},{b}".format(a='A',b='B')) # A,B
print("{0[0]},{0[1]}".format(['A','B'])) # A,B
print("{:.2f}".format(3.1415926))    # 3.14

# 对象的format格式化输出
class A(object):
    def __init__(self, value):
        self.value = value
a = A(6)
print('{0.value}'.format(a)) # 6
```

## 2.常用函数

```python
l = [-2, -1, 0, 1, 2]
max(l)               # 2
min(l)               # -2
len(l)               # 5 
sum(l)               # 0    

abs(-1)              # 1      绝对值
pow(2, 5)            # 32     开方
divmod(5, 2)         # (2, 1) (商,余数)
round(1.25361, 3)    # 1.254  保留小数
round(1627731, -1)   # 1627730
hash('B')            # 8720545829485517778 返回对象的哈希值

a = complex(1, 2)    # 或 a = 1 + 2j
a.real               # 1.0
a.imag               # 2.0
a.conjugate()        # (1-2j) 

for i in range(5):
    print(i)  # 0 1 2 3 4
for i in range(1, 5):
    print(i)  # 1 2 3 4
for i in range(1, 5, 2):
    print(i)  # 1 3
```

## 3.函数式编程

```Python
# map映射
def func(x):
    return x * x  
for i in map(func, [1, 2, 3]):
    print(i) # 1 4 9

for i in map(lambda x: x*x, [1, 2, 3]):
    print(i)  # 1 4 9
for i in map(lambda x,y: x+y, [1,3,5], [2,4,6]):
    print(i)  # 3 7 11

# filter过滤
for i in filter(lambda e: e%2, [1, 2, 3]):  
    print(i)  # 1 3

# sorted排序
print(sorted([1, 5, 2], reverse=True))              
# [5, 2, 1]
print(sorted([('b', 2), ('a', 1)], key=lambda x:x[0]))   
# [('a', 1), ('b', 2)]

# zip打包
l1 = ['a','b','c']
l2 = [1,2,3]
for i in zip(l1, l2):
    print(i)  # ('a', 1) ('b', 2) ('c', 3)       

# zip解压
for i in zip(*zip(l1, l2)):
    print(i)  # ('a', 'b', 'c') (1, 2, 3)

# reduce累积
from functools import reduce
def add(x, y):
    return x + y
print(reduce(add, [1, 3, 5])) # 9 

# 表达式执行
eval("1+2*3")  # 7 

r = compile("3*4+5",'','eval')
eval(r)       # 17

r = compile("print('hello,world')", "<string>", "exec")
exec(r)       # hello,world
```

## 4.类型转换

```python
int('123')     # 123
float('123.4') # 123.4
str(123)       # '123' 
bool(2)        # True

bin(2)         # 0b10 二进制
oct(8)         # 0o10 八进制
hex(16)        # 0x10 十六进制

bytes(1)       # b'\x00'
ord('a')       # 97 
chr(97)        # a

list((1,2,3))        # [1, 2, 3]
set([1,2,3,3])       # {1, 2, 3}
frozenset([1,2,3,3]) # frozenset({1, 2, 3})   

dict([('A', 1), ('B', 2)])     # {'A': 1, 'B': 2}
dict(zip(['A', 'B'], [1, 2]))  # {'A': 1, 'B': 2}

# 拷贝
import copy 
l1 = ['a', 'b', 'c', 'd']
l2 = copy.copy(l1)       
print(id(l1), id(l2))

l3 = [[1,2,3],4,5,6]
l4 = copy.deepcopy(l1)   # 列表嵌套列表时使用深拷贝
l5 = copy.copy(l3)    
print(id(l3), id(l4), id(l5))
```

## 5.自省函数

```python
# 对象自省
type(2)       # <class 'int'>
id(2)         # 140731323023664 内存地址
dir([str])    # 返回对象的属性、方法列表 
help('sys')   # 返回对象的帮助文档
locals()      # 返回当前局部变量的字典
globals()     # 返回当前全局变量的字典  是指定义它们的模块，而不是调用它们的模块
vars()        # 返回对象属性和属性值的字典

all([0,1,2])  # False 所有元素为真返回True
any([0,1,2])  # True  任一元素为真返回True

callable(2)                  # False 是否可调用
isinstance(2,int)            # True  是否是其实例
isinstance(2,(str,int,list)) # True  满足一项返回True
issubclass(str, int)         # False 是否是其子类

# 类的属性
class Student(object):
    def __init__(self, name):
        self.name = name

s = Student('Tom')
getattr(s, 'name')    # Tom
#getattr(s, 'age')    # 属性不存在触发AttributeError异常              
getattr(s, 'age', 5)  # 属性age不存在，但会返回默认值5
hasattr(s, 'age')     # False，查询属性是否存在  
setattr(s, 'age', 5)  # 设置属性 age 值
delattr(s, 'age')     # 删除属性
```