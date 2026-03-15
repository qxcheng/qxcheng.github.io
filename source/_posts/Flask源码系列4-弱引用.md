---
categories:
- - flask
date: 2024-04-19 10:15:00
description: "本篇文章介绍Python中弱引用的原理和使用。"
id: '172'
tags:
- python
- web框架
title: Flask源码系列4-弱引用
---


本篇文章介绍Python中弱引用的原理和使用。

flask源码：

```python
class Flask(App):
    ...
    def __init__(self, ...):
        ...
        if self.has_static_folder:
            assert (
                bool(static_host) == host_matching
            ), "Invalid static_host/host_matching combination"
            # Use a weakref to avoid creating a reference cycle between the app
            # and the view function (see #3761).
            self_ref = weakref.ref(self)
            self.add_url_rule(
                f"{self.static_url_path}/<path:filename>",
                endpoint="static",
                host=static_host,
                view_func=lambda **kw: self_ref().send_static_file(**kw),  # type: ignore # noqa: B950
            )
```

## 1.原理

在Python中，弱引用是一种特殊的引用，它允许一个对象被引用，而不阻止它被垃圾回收器（GC）回收。这与普通的引用（强引用）不同，强引用会阻止对象被回收。弱引用在某些场合特别有用，尤其是在处理缓存或循环引用时。

1.  垃圾回收：在Python中，当一个对象的引用计数降到零时，垃圾回收器会回收它。对于强引用，每个引用都增加对象的引用计数。而弱引用不增加引用计数。
2.  对象生命周期：当一个对象只剩下弱引用时，它可以被垃圾回收器回收。这意味着弱引用不会阻止对象的内存被释放。
3.  访问被引用对象：通过弱引用，你可以访问被引用的对象，但如果该对象已经被回收，访问它会得到 `None`。

**使用弱引用的场景：**

1.  缓存机制：弱引用常用于缓存应用，因为它们允许对象在不再需要时被自动回收。
2.  防止循环引用：循环引用（两个对象互相引用）可能导致内存泄漏，因为它们的引用计数永远不会达到零。使用弱引用可以打破循环，从而使得垃圾回收器能够回收这些对象。

## 2.使用

```python
import weakref

class Data:
    def __init__(self, key):
        self.value = key * 2

# 数据缓存
class Cacher:
    def __init__(self):
        self.pool = {}

    def get(self, key):
        r = self.pool.get(key)
        if r:
            # 调用弱引用对象，即可找到指向的对象
            # 删除变量后，再次调用弱引用对象返回None
            data = r()
            if data:
                print(f'cache hits {key}, value is {data.value}')
                return data

        print(f'cache set {key}')
        data = Data(key)
        # 创建一个指向该数据的弱引用
        self.pool[key] = weakref.ref(data)  
        return data

if __name__ == '__main__':
    cacher = Cacher()

    data = cacher.get(1)
    data2 = cacher.get(1)
    print(data is data2)

    del data
    del data2
    data3 = cacher.get(1)

$
cache set 1
cache hits 1, value is 2
True
cache set 1
```

1.  data通过创建赋值得到，引用计数+1
2.  data2通过调用弱引用对象得到，引用计数+1
3.  在删除data、data2后，引用计数变成0触发垃圾回收，data3重新通过创建赋值得到

## 3.弱引用映射类

weakref提供了弱引用映射类，可以简化代码：

*   _weakref.WeakKeyDictionary_ ，键只保存弱引用的映射类（一旦键不再有强引用，键值对条目将自动消失）；
*   _weakref.WeakValueDictionary_ ，值只保存弱引用的映射类（一旦值不再有强引用，键值对条目将自动消失）；

```python
import weakref

class Data:
    def __init__(self, key):
        self.value = key * 2

# 数据缓存
class Cacher:
    def __init__(self):
        self.pool = weakref.WeakValueDictionary()

    def get(self, key):
        data = self.pool.get(key)
        if data:
            print(f'cache hits {key}, value is {data.value}')
            return data
        print(f'cache set {key}')
        self.pool[key] = data = Data(key)
        return data

if __name__ == '__main__':
    cacher = Cacher()

    data = cacher.get(1)
    data2 = cacher.get(1)
    print(data is data2)

    del data
    del data2
    data3 = cacher.get(1)
```

## 4.使用限制

*   基本的 list 和 dict 实 例不能作为所指对象，但是它们的子类可以。
*   int 和 tuple 实例不能作 为弱引用的目标，甚至它们的子类也不行。

```python
import weakref

i = 1

ref_i = weakref.ref(i)

$
Traceback (most recent call last):
  File "C:\test\main.py", line 5, in <module>
    ref_i = weakref.ref(i)
TypeError: cannot create weak reference to 'int' object
```

```python
import weakref

class MyList(list):
    """list的子类，实例可以作为弱引用的目标"""

a_list = MyList(range(10))

wref_to_a_list = weakref.ref(a_list)
```