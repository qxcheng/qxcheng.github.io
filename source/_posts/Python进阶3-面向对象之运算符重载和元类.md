---
categories:
- - Python
date: 2024-04-16 16:27:14
description: 'Python进阶3-面向对象之运算符重载和元类'
id: '96'
tags:
- python
title: Python进阶3-面向对象之运算符重载和元类
---


## 1.常用运算符

```python
class Vector(object):
    # 构造方法 先于init调用
    # __new__ 方法也可以返回其他类的实例，此时，解释器不会调用 __init__ 方法。
    def __new__(cls, *args, **kwargs):
        print("call __new__")
        return object.__new__(cls, *args, **kwargs)  # 返回实例对象

    def __init__(self, x, y):
        self.x = x
        self.y = y
        self.uid = x + y

    # print(vector)时显示  str(vector)  '%s' % vector
    def __str__(self):
        return 'x: %s, y: %s' % (self.x, self.y)

    # 对象vector的标准表示,也就是vector是如何创建的  repr(vector)  '%r' % vector
    def __repr__(self):
        return 'Vector(%s, %s)' % (self.x, self.y)

    # 重载加号运算符，类似的：sub  mul  div  mod  pow
    # eq:== ne:!= lt:< gt:>
    def __add__(self, other):
        return Vector(self.x+other.x, self.y+other.y)

    # 实例直接调用自身
    def __call__(self):
        print('Called by myself!')

    # 在对象被删除后引用计数为0时触发
    def __del__(self):
        self.conn.close()

    # 访问不存在的属性时调用: .调用对象属性时触发 getattr()也可触发
    def __getattr__(self, name):
        if name == 'adult':
            if self.age > 1.0: return 1
            else: return False
        else: raise AttributeError(name)

    # 通过「.」设置属性或 `setattr(key, value)` 设置属性时调用
    def __setattr__(self, key, value):
        super(Vector, self).__setattr__(key, value)

    # 删除某个属性时调用
    def __delattr__(self, key):
        super(Vector, self).__delattr__(key)

    def __getattribute__(self, key):
        """访问任意属性/方法都首先经过这里"""
        if key == 'hello':
            return self.say
        if key == 'world':
            raise AttributeError  # 触发异常后会继续调用__getattr__

        return super(Vector, self).__getattribute__(key)

    def say(self):
        return 'hello'

    # 用来表示实例对象的唯一标识，配合 `__eq__` 方法，可以判断两个对象是否相等
    # 在 `set` 中存放这些对象，也会根据这两个方法进行去重操作
    def __hash__(self):
        return self.uid

    def __eq__(self, other):
        return self.uid == other.uid

v1 = Vector(1, 1)
print(v1)      # Vector(1, 1)

v2 = Vector(2, 2)
print(v1 + v2) # Vector(3, 3)

v2()           # Called by myself!
```

## 2.容器运算符

```python
class IntList(object):
    def __init__(self, *args):
        if all(map(lambda x:isinstance(x, int), args)):
            self._list = list(args)
        else:
            print("所有传入的参数必须是整数！")

    def __getitem__(self, index):
        return self._list[index]

    def __setitem__(self, index, value):
        self._list[index] = value

    def __delitem__(self, index):
        del self._list[index]

    def __len__(self):
        print("调用__len__方法")
        return len(self._list)

    def __contains__(self, key):
        # 元素是否在自定义list中 in
        return key in self.values

    def __reversed__(self):
        # 反转 reversed()
        return list(reversed(self.values))

l = IntList(1,2,3,4,5)
print(len(l))
# 调用__len__方法
# 5

print(l[0]) # 1

l[0] = -1
print(l[0]) # -1

del l[0]
print(l[0]) # 2
```

切片返回实例：

```python
def __len__(self):
    return len(self._components)

def __getitem__(self, index):
    cls = type(self) 
    if isinstance(index, slice): 
        return cls(self._components[index]) 
    elif isinstance(index, numbers.Integral): 
        return self._components[index] 
    else:
        msg = '{cls.__name__} indices must be integers'
        raise TypeError(msg.format(cls=cls)) 
```

## 3.序列化

```python
class Person(object):

    def __init__(self, name, age, birthday):
        self.name = name
        self.age = age
        self.birthday = birthday

    def __getstate__(self):
        # 执行 pick.dumps 时 忽略 age 属性
        return {
            'name': self.name,
            'birthday': self.birthday
        }

    def __setstate__(self, state):
        # 执行 pick.loads 时 忽略 age 属性
        self.name = state['name']
        self.birthday = state['birthday']

person = Person('zhangsan', 20, date(2017, 2, 23))
pickled_person = pickle.dumps(person) # __getstate__

p = pickle.loads(pickled_person) # __setstate__
print(p.name, p.birthday)

print(p.age) # AttributeError: 'Person' object has no attribute 'age'
```

## 4.元类

```Python
#1 type()动态创建类
def myfunc(self, name='Tom'):
     print('Hello, %s!' % self.name)

Student = type('Student', (object,), dict(name='Jack', func=myfunc)) 
#参数分别为：class名称、继承的父类集合、方法名称与函数绑定
s = Student()
s.func() # Hello, Jack!

#2 metaclass元类
class ListMetaclass(type): 
    def __new__(cls, name, bases, attrs):
        # __new__ 是在__init__之前被调用的特殊方法
        # __new__是用来创建对象并返回之的方法,  而__init__只是用来将传入的参数初始化给对象
        attrs['doc'] = 'A metaclass!'  # 为类动态添加属性
        attrs['add'] = lambda self, value: self.append(value)  # 为类动态添加方法

        for name, value in attrs.items():
            print("name=%s and value=%s" % (name,value))  # 打印所有类属性出来
            if not name.startswith("__"):
               attrs[name.upper()] = value
               print("name.upper()=", name.upper())
               print("value=",value)

        return type.__new__(cls, name, bases, attrs)
        #return type(class_name, class_parents, new_attr)

# 使用ListMetaclass来定制类
class MyList(list, metaclass=ListMetaclass):
    name = "MYLIST"

L = MyList()
L.add(1)  
print(L)     # [1]
print(L.doc) # A metaclass!
```

## 5.单例模式

```Python
#1 装饰器实现
def Singleton(cls):
    _instance = {}

    def _singleton(*args, **kargs):
        if cls not in _instance:
            _instance[cls] = cls(*args, **kargs)
        return _instance[cls]

    return _singleton

@Singleton
class A(object):
    a = 1

    def __init__(self, x=0):
        self.x = x

a1 = A(2)
a2 = A(3)

#2 类实现
class Singleton(object):
    def __init__(self):
        pass

    def __new__(cls, *args, **kwargs):
        if not hasattr(Singleton, "_instance"): # 反射
            Singleton._instance = object.__new__(cls)
        return Singleton._instance

obj1 = Singleton()
obj2 = Singleton()
print(id(obj1) == id(obj2)) # True  

#3 多线程
import threading
class Singleton(object):
    _instance_lock = threading.Lock()

    def __init__(self):
        pass

    def __new__(cls, *args, **kwargs):
        if not hasattr(Singleton, "_instance"):
            with Singleton._instance_lock:
                if not hasattr(Singleton, "_instance"):
                    Singleton._instance = object.__new__(cls)  
        return Singleton._instance

obj1 = Singleton()
obj2 = Singleton()
print(obj1,obj2)

def task(arg):
    obj = Singleton()
    print(obj)

for i in range(10):
    t = threading.Thread(target=task,args=[i,])
    t.start()
```