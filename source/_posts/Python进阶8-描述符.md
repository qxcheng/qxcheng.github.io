---
categories:
- - Python
date: 2024-04-17 12:17:04
description: 'Python进阶8-描述符'
id: '109'
tags:
- python
title: Python进阶8-描述符
---


## 描述符
- 描述符必须是一个类属性
- `__getattribute__` 是查找一个属性（方法）的入口
- `__getattribute__` 定义了一个属性（方法）的查找顺序：数据描述符、实例属性、类属性、非数据描述符、父类、调用` __getattr__`方法
- 如果我们重写了 `__getattribute__` 方法，会阻止描述符的调用
- 所有方法其实都是一个非数据描述符，因为它定义了 `__get__`

```Python
class Validator:

    def __init__(self):
        self.data = {}

    def __get__(self, obj, objtype=None):
        return self.data[obj]

    def __set__(self, obj, value):
        # 校验通过后再赋值 obj为Person对对象
        self.validate(value)
        self.data[obj] = value

    def validate(self, value):
        pass

class Number(Validator):

    def __init__(self, minvalue=None, maxvalue=None):
        super(Number, self).__init__()
        self.minvalue = minvalue
        self.maxvalue = maxvalue

    def validate(self, value):
        if not isinstance(value, (int, float)):
            raise TypeError(f'Expected {value!r} to be an int or float')
        if self.minvalue is not None and value < self.minvalue:
            raise ValueError(
                f'Expected {value!r} to be at least {self.minvalue!r}'
            )
        if self.maxvalue is not None and value > self.maxvalue:
            raise ValueError(
                f'Expected {value!r} to be no more than {self.maxvalue!r}'
            )

class String(Validator):

    def __init__(self, minsize=None, maxsize=None):
        super(String, self).__init__()
        self.minsize = minsize
        self.maxsize = maxsize

    def validate(self, value):
        if not isinstance(value, str):
            raise TypeError(f'Expected {value!r} to be an str')
        if self.minsize is not None and len(value) < self.minsize:
            raise ValueError(
                f'Expected {value!r} to be no smaller than {self.minsize!r}'
            )
        if self.maxsize is not None and len(value) > self.maxsize:
            raise ValueError(
                f'Expected {value!r} to be no bigger than {self.maxsize!r}'
            )

class Person:

    # 定义属性的校验规则 内部用描述符实现
    name = String(minsize=3, maxsize=10)
    age = Number(minvalue=1, maxvalue=120)

    def __init__(self, name, age):
        self.name = name
        self.age = age

# 属性符合规则
p1 = Person('zhangsan', 20)
print(p1.name, p1.age)

# 属性不符合规则
p2 = Person('a', 20)
# ValueError: Expected 'a' to be no smaller than 3
p3 = Person('zhangsan', -1)
# ValueError: Expected -1 to be at least 1
```


```python
class Quantity: 
    def __init__(self, storage_name):
        self.storage_name = storage_name 
        
    def __set__(self, instance, value): 
        if value > 0:
            instance.__dict__[self.storage_name] = value  # 对于同名属性 不能使用 setattr 否则会递归调用__set__
        else:
            raise ValueError('value must be > 0')

class LineItem:
    weight = Quantity('weight') 
    price = Quantity('price') 

    def __init__(self, description, weight, price): 
        self.description = description
        self.weight = weight
        self.price = price

    def subtotal(self):
        return self.weight * self.price

```

不要参数
```python
class Quantity:
    __counter = 0 

    def __init__(self):
        cls = self.__class__ 
        prefix = cls.__name__
        index = cls.__counter
        self.storage_name = '_{}#{}'.format(prefix, index)
        cls.__counter += 1 

    def __get__(self, instance, owner): 
        return getattr(instance, self.storage_name) 

    def __set__(self, instance, value):
        if value > 0:
            setattr(instance, self.storage_name, value) 
        else:
            raise ValueError('value must be > 0')

class LineItem:
    weight = Quantity() 
    price = Quantity() 

    def __init__(self, description, weight, price): 
        self.description = description
        self.weight = weight
        self.price = price

    def subtotal(self):
        return self.weight * self.price

```

验证
```python
import abc

class AutoStorage:
    __counter = 0 

    def __init__(self):
        cls = self.__class__ 
        prefix = cls.__name__
        index = cls.__counter
        self.storage_name = '_{}#{}'.format(prefix, index)
        cls.__counter += 1 

    def __get__(self, instance, owner): 
        if instance is None:
            return self
        else:
            return getattr(instance, self.storage_name) 

    def __set__(self, instance, value):
        setattr(instance, self.storage_name, value) 

class Validated(abc.ABC, AutoStorage): 
    def __set__(self, instance, value):
        value = self.validate(instance, value) 
        super().__set__(instance, value) 

    @abc.abstractmethod
    def validate(self, instance, value): 
        """return validated value or raise ValueError"""

class Quantity(Validated): 
    """a number greater than zero"""
    def validate(self, instance, value):
        if value <= 0:
            raise ValueError('value must be > 0')
        return value
    
class NonBlank(Validated):
    """a string with at least one non-space character"""
    def validate(self, instance, value):
        value = value.strip()
        if len(value) == 0:
            raise ValueError('value cannot be empty or blank')
        return value

import model_v5 as model 
class LineItem:
    description = model.NonBlank() 
    weight = model.Quantity()
    price = model.Quantity()

    def __init__(self, description, weight, price):
        self.description = description
        self.weight = weight
        self.price = price

    def subtotal(self):
        return self.weight * self.price
```
