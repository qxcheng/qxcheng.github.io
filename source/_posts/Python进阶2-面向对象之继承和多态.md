---
categories:
- - Python
date: 2024-04-16 16:20:18
description: 'Python进阶2-面向对象之继承和多态'
id: '94'
tags:
- python
title: Python进阶2-面向对象之继承和多态
---


## 1.单继承

```Python
#1 单继承
class Base(object):
    def __init__(self):
        print('Base')

class A(Base):
    def __init__(self):   
        # Student.__init__(self, name)
        super().__init__() # 调用super()会得到一个用来引用父类属性的对象，且严格按照
                               # MRO规定的顺序向后查找。即使没有直接继承关系，super()仍然
                               # 会按照MRO继续往后查找                        
        print('A')

b = B()
# Base
# A
```

## 2.多继承

```python
'''
子类会先于父类被检查
多个父类会根据它们在列表中的顺序被检查
如果对下一个类存在两个合法的选择，选择第一个父类  
'''
class Base:
    def __init__(self):
        print('Base')

class A(Base):
    def __init__(self):
        super().__init__()
        print('A')

class B(Base):
    def __init__(self):
        super().__init__()
        print('B')

class C(A,B):
    def __init__(self):
        super().__init__()  # Only one call to super() here
        print('C')

c = C()
# Base
# B
# A
# C
C.__mro__ #所有基类的线性顺序表
```

## 3.继承内置类型

内置类型（使用 C 语言编写）不会调用用户定义的类覆盖的特殊方法。 例如，dict 的子类覆盖的 **getitem**() 方法不会被内置类型的 get() 方法调用。内置类型 dict 的 **init** 和 **update** 方法会忽略我们覆盖的 **setitem** 方法

不要子类化内置类型，用户自己定义的类应该继承 collections 模块中的类，例如 UserDict、UserList 和 UserString

```python
class DoppelDict(dict):
    def __setitem__(self, key, value):
        super().__setitem__(key, [value] * 2) 

>>> dd = DoppelDict(one=1) 
>>> dd
{'one': 1}
>>> dd['two'] = 2 
>>> dd
{'one': 1, 'two': [2, 2]}
>>> dd.update(three=3) 
>>> dd
{'three': 3, 'one': 1, 'two': [2, 2]}
```

## 4.抽象类和多态

```Python
import abc

# 指定metaclass属性将类设置为抽象类，抽象类本身只是用来约束子类的，不能被实例化
class Animal(metaclass=abc.ABCMeta):
    @abc.abstractmethod # 该装饰器限制子类必须定义有一个名为talk的方法
    def talk(self): # 抽象方法中无需实现具体的功能
        pass

class Cat(Animal): # 但凡继承Animal的子类都必须遵循Animal规定的标准
    def talk(self):
        pass

class Dog(Animal): #动物的形态之二:狗
    def talk(self):
        print('汪汪汪')      

cat=Cat() # 若子类中没有一个名为talk的方法则会抛出异常TypeError，无法实例化
dog=Dog()

cat.talk()
dog.talk()

def Talk(animal):
    animal.talk()

Talk(cat)    # 鸭子类型
Talk(dog)  
```