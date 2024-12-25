---
categories:
- - flask
date: 2024-04-19 10:18:56
description: "本篇文章介绍Python中描述符的原理和使用。"
id: '178'
tags:
- python
- web框架
title: Flask源码系列6-描述符
---


本篇文章介绍Python中描述符的原理和使用。

flask源码：

```python
# config.py
class ConfigAttribute:
    """Makes an attribute forward to the config"""

    def __init__(self, name: str, get_converter: t.Callable  None = None) -> None:
        self.__name__ = name
        self.get_converter = get_converter

    def __get__(self, obj: t.Any, owner: t.Any = None) -> t.Any:
        if obj is None:
            return self
        rv = obj.config[self.__name__]
        if self.get_converter is not None:
            rv = self.get_converter(rv)
        return rv

    def __set__(self, obj: t.Any, value: t.Any) -> None:
        obj.config[self.__name__] = value

# sansio/app.py
class App(Scaffold):
    ...
    secret_key = ConfigAttribute("SECRET_KEY")
```

## 1.原理

在Python中，描述符是一种实现了特定方法的对象，这些方法是 `__get__()`、`__set__()` 和 `__delete__()`。描述符提供了一种强大的方法来重新定义属性的访问、设置和删除行为。它们是Python属性访问机制的基础，并且被用于实现属性、方法、静态方法和类方法。

1.  **描述符协议**：描述符是实现了描述符协议的对象。描述符协议包括以下方法：
    
    *   `__get__(self, obj, type=None)`：访问属性时被调用。
    *   `__set__(self, obj, value)`：设置属性时被调用。
    *   `__delete__(self, obj)`：删除属性时被调用。
2.  **类型**：描述符分为两类：
    
    *   **数据描述符**：同时实现了 `__get__()` 和 `__set__()` 方法。
    *   **非数据描述符**：只实现了 `__get__()` 方法。
3.  **属性访问优先级**：当使用点号访问属性时，Python解释器会根据以下规则查找属性：
    
    *   如果对象的类定义了与属性同名的数据描述符，则调用数据描述符的 `__get__()` 方法。
    *   如果对象本身的实例字典（`obj.__dict__`）中存在该属性，则直接返回该属性。
    *   如果对象本身存在与属性同名的类属性，则直接返回该属性。
    *   如果对象的类定义了与属性同名的非数据描述符，则调用非数据描述符的 `__get__()` 方法。
    *   如果找不到属性，则查找对象的类的父类，重复上述过程。
    *   `__getattr__` 方法：如果以上步骤都未能定位到属性，并且类定义了 `__getattr__` 方法，那么将调用 `__getattr__(attr)`。如果未定义`__getattr__` 方法，那么将继续查找父类的`__getattr__` 方法。
    *   抛出 `AttributeError`：如果所有这些尝试都失败了，将会抛出 `AttributeError` 异常。

## 2.使用

下面是一个用描述符实现数据验证的示例：

```python
class Validator:
    _index = 0

    def __init__(self):
        cls = self.__class__
        self.attr_name = '_{}#{}'.format(cls.__name__, cls._index)
        cls._index += 1 

    def __get__(self, instance, owner):
        # instance代表托管的实例
        # owner代表托管实例的类的类型
        if instance is None:
            return self
        else:
            # 如果这里的attr_name和托管实例的属性名一样，则不能用getattr方法，会造成递归调用，
            # 而是应该使用 instance.__dict__[self.attr_name] 直接访问属性
            return getattr(instance, self.attr_name) 

    def __set__(self, instance, value):
        # instance代表托管的实例
        # 校验通过后再赋值
        self.validate(value)
        setattr(instance, self.attr_name, value) 

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
                f'Expected {value!r} to be no smaller than {self.minvalue!r}'
            )
        if self.maxvalue is not None and value > self.maxvalue:
            raise ValueError(
                f'Expected {value!r} to be no bigger than {self.maxvalue!r}'
            )

class String(Validator):

    def __init__(self, minlength=None, maxlength=None):
        super(String, self).__init__()
        self.minlength = minlength
        self.maxlength = maxlength

    def validate(self, value):
        if not isinstance(value, str):
            raise TypeError(f'Expected {value!r} to be an str')
        if self.minlength is not None and len(value) < self.minlength:
            raise ValueError(
                f'Expected {value!r} to be no shorter than {self.minlength!r}'
            )
        if self.maxlength is not None and len(value) > self.maxlength:
            raise ValueError(
                f'Expected {value!r} to be no longer than {self.maxlength!r}'
            )

class Person:

    # 定义属性的校验规则 
    name = String(minlength=2, maxlength=10)
    age = Number(minvalue=1, maxvalue=120)

    def __init__(self, name, age):
        self.name = name
        self.age = age

# 属性符合规则
p1 = Person('Alice', 18)
print(p1.name, p1.age)
print(p1.__dict__) 
# Alice 18
# {'_String#0': 'Alice', '_Number#0': 18}

# 属性不符合规则
# p2 = Person('A', 20)
# ValueError: Expected 'A' to be no shorter than 2

# p3 = Person('Tom', -1)
# ValueError: Expected -1 to be no smaller than 1
```

## 3.类的方法

类的方法用描述符实现简易版：

```python
class method:
    def __init__(self, method):
        self.method = method

    def __get__(self, obj, cls=None):
        def wrapper(*args, **kwargs):
            return self.method(obj, *args, **kwargs)
        return wrapper

def count(self, num):
    self.num += num
    return self.num

# 使用示例
class MyClass:
    count = method(count)

    def __init__(self):
        self.num = 0

a = MyClass()
print(a.count(1))  # 1
```

## 4.classmethod

classmethod用描述符实现简易版：

```python
class classmethod:
    def __init__(self, method):
        self.method = method

    def __get__(self, obj, cls=None):
        if cls is None:
            cls = type(obj)
        def wrapper(*args, **kwargs):
            return self.method(cls, *args, **kwargs)
        return wrapper

# 使用示例
class MyClass:
    _counter = 0

    def __init__(self):
        MyClass._counter += 1

    @classmethod
    def count(cls):
        return cls._counter

a = MyClass()
print(MyClass.count())  # 1
```

## 5.staticmethod

staticmethod用描述符实现简易版：

```python
class staticmethod:
    def __init__(self, method):
        self.method = method

    def __get__(self, obj, cls=None):
        def wrapper(*args, **kwargs):
            return self.method(*args, **kwargs)
        return wrapper

# 使用示例
class MyClass:

    @staticmethod
    def count(num, base=100):
        return num + base

print(MyClass().count(1))  # 101
print(MyClass.count(2))    # 102
```

## 6.property

property用描述符实现简易版：

```python
class property:

    def __init__(self, fget=None, fset=None, fdel=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel)

class C(object):
    def __init__(self):
        self._x = 'Tom'

    @property
    def x(self):
        return self._x

    @x.setter
    def x(self, value):
        self._x = value

c = C()
print(c.x)  # Tom

c.x = 'Tony'
print(c.x)  # Tony
```

1.  @property装饰器将x变成property的实例，即一个数据描述符，同时将装饰的方法赋值给了self.fget。此时x具备了setter方法；
2.  @x.setter装饰器将x变成property的实例，即一个数据描述符，同时将装饰的方法赋值给了self.fset，并保留了self.fget和self.fdel属性。此时，x具备了self.fget和self.fset方法，数据描述符能正常设置和取值。