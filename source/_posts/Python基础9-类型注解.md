---
categories:
- - Python
date: 2024-04-16 14:10:19
description: 'Python基础9-类型注解'
id: '88'
tags:
- python
title: Python基础9-类型注解
---


## 1.变量注解

```python
a: int = 1
b: float = 1.5
c: str = 'jack'
print(a, b, c)  # 1 1.5 jack

d: list = [0]
e: dict = {0:0}
f: set = {2,1,1}
g: frozenset = {1,3,2}
h: tuple = (1,2,3)
print(d, e, f, g, h)  # [0] {0: 0} {1, 2} {1, 2, 3} (1, 2, 3)
```

## 2.函数注解

注解不会做任何处理，只是存储在函数的 **annotations** 属性（一个字典）中

```python
from typing import Optional, Union, NoReturn, Callable, Any, Literal

# 参数是int 返回值是int
def square(num: int) -> int:
    return num ** 2

# 参数是str 返回值是None
def func(name: str) -> None:
    print(name)

# NoReturn: 函数没有返回值（注：python函数默认隐式返回None）
def func(name: str) -> NoReturn:
    raise RuntimeError('error')

# Optional: 函数可返回str或None
def func(name: str) -> Optional[str]:
    if name:
        return name

# Union: 函数可返回str或int
def func(name: str) -> Union[str, int]:
    if name:
        return name
    else:
        return 0

# Callable: 参数是可调用的类型（函数 或 实现__call__方法类的实例）
# Any: 函数可返回任意类型
def func(myfunc: Callable) -> Any:
    myfunc('Hello')
func(print)

# Literal: 参数是定义好的字面量
Operator = Literal['+', '-', '*', '/']
def func(operator: Operator) -> None:
    print(operator)
func('%')  # 类型检查不通过
func('+')  # 类型检查通过
```

## 3.容器注解

```python
from typing import List, Dict, Tuple, Sequence

# List, Dict, Tuple
def func(names: List[str], ages: Dict[str, int]) -> Tuple[str, int]:
    return ('jack', 20)

# Sequence：不对容器的类型做具体要求
def max_age(ages: Sequence[int]):
    return max(ages)
print(max_age([18,28,38]))
```

## 4.类型别名

```python
from typing import Tuple, NewType

# 类型别名
Point = Tuple[int, int, int] 

def func(point: Point):
    print(point)

func(point=(2, 3, 4))

# 创建新类型：需传入新类型的实例
NewPoint = NewType('NewPoint', Tuple[int, int, int])

def new_func(point: NewPoint):
    print(point)

new_func(point=(2, 3, 4))            # 类型检查不通过
new_func(point=NewPoint((2, 3, 4)))  # 类型检查通过
```

## 5.类注解

```python
from typing import  Optional, Protocol

# 使用未定义的类时，用字符串表示即可
class Node():
    def __init__(self, parent:Optional['Node']=None):
        self.parent = parent

node: Node = Node()

# Protocol：需实现协议定义的方法
class Proto(Protocol):
    def func(self):
        pass

class A():
    def func(self):
        pass

class B():
    pass

a: Proto = A()  # 类型检查通过
b: Proto = B()  # 类型检查不通过
```

## 6.泛型

```python
from typing import TypeVar

# 泛型的具体类型是str或int
T = TypeVar('T', str, int)

def func(a: T, b: T) -> T:
    return a + b

func(1, 2)      # 类型检查通过
func('1', '2')  # 类型检查通过
# func(1, 'a')  # 类型检查不通过：只能为其一

# 泛型的具体类型没有限制
from typing import TypeVar, List

TL = TypeVar('TL')

def func(l: List[TL], value: TL) -> None:
    l.append(value)

l = [1, 2, 3]  # 若这里 l=[]，下面的都能通过 
func(l, 1)     # 类型检查通过
func(l, 'a')   # 类型检查不通过
```