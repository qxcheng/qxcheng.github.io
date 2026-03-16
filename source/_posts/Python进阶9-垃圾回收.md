---
categories:
- - Python
date: 2024-04-17 13:17:04
description: 'Python进阶9-垃圾回收'
id: '110'
tags:
- python
title: Python进阶9-垃圾回收
---

## 垃圾回收

1.引用计数：被引用为0时，立即回收当前对象

2.标记-清除：用于处理循环引用，只有容器对象（list、dict、tuple，instance）才会出现循环引用的情况 root链表：循环引用时，先互相-1，然后删除a会导致b引用计数-1， 删除b导致a引用计数-1， 都为0 放入unreachable链表 unreachable链表：只删除a 由于b引用了a 导致被放入root链表

3.分代回收

0代 1代 2代 对应三个链表 新创建的对象都会分配在年轻代，年轻代链表的总数达到上限时，Python垃圾收集机制就会被触发，把那些可以被回收的对象回收掉，而那些不会回收的对象就会被移到中年代去，依此类推，老年代中的对象是存活时间最久的对象，甚至是存活于整个系统的生命周期内。



**引用计数**
```python
age = 18  #  18的引用计数为1

m=age  # 变量值18的引用计数为2

age=10 变量值18的引用计数为1 

del m 变量值18的引用计数为0

```

当使用某个引用作为参数，传递给getrefcount()时，参数实际上创建了一个临时的引用。因此，getrefcount()所得到的结果，会比期望的多1。
```python
from sys import getrefcount

a = [1, 2, 3]
print(getrefcount(a))  # 2

b = a
print(getrefcount(b))  # 3


```

用objgraph包来绘制其引用关系
```python
sudo apt-get install xdot
sudo pip install objgraph

x = [1, 2, 3]
y = [x, dict(key1=x)]
z = [y, (x, y)]

import objgraph
objgraph.show_refs([z], filename='ref_topo.png')

```

当Python运行时，会记录其中分配对象(object allocation)和取消分配对象(object deallocation)的次数。当两者的差值高于某个阈值时，垃圾回收才会启动。我们可以通过gc模块的get_threshold()方法，查看该阈值:
```python
import gc
print(gc.get_threshold())
# (700, 10, 10)
每10次0代垃圾回收，会配合1次1代的垃圾回收；而每10次1代的垃圾回收，才会有1次的2代垃圾回收。
700即是垃圾回收启动的阈值。可以通过gc中的set_threshold()方法重新设置。
gc.set_threshold(700, 10, 5)

手动启动垃圾回收，即使用gc.collect()。



```

Python将所有的对象分为0，1，2三代。所有的新建对象都是0代对象。当某一代对象经历过垃圾回收，依然存活，那么它就被归入下一代对象。垃圾回收启动时，一定会扫描所有的0代对象。如果0代经过一定次数垃圾回收，那么就启动对0代和1代的扫描清理。当1代也经历了一定次数的垃圾回收后，那么会启动对0，1，2，即对所有对象进行扫描。


**循环引用问题**
```Python
l1=['xxx']    # 列表1的引用计数变为1   
l2=['yyy']    # 列表2的引用计数变为1   
l1.append(l2) # 列表2的引用计数变为2
l2.append(l1) # 列表1的引用计数变为2
```
![[pasted_image003_kL7F5qJy_U.png]]

```Python
del l1  # 列表1的引用计数变为1
del l2  # 列表2的引用计数变为1
此时两个列表不再被任何其他对象关联引用, 但两个列表的引用计数均不为0
```

![[pasted_image004_0XV9UQwMJK.png]]

**解决方案：标记-清除**
1、标记 遍历所有的GC Roots对象(栈区中的所有内容或者线程都可以作为GC Roots对象），然后将所有GC Roots的对象可以直接或间接访问到的对象标记为存活的对象，其余的均为非存活对象，应该被清除。 2、清除 清除的过程将遍历堆中所有的对象，将没有标记的对象全部清除掉。

这样在启用标记清除算法时，从栈区出发，没有任何一条直接或间接引用可以访达l1与l2，于是l1与l2都没有被标记为存活，二者会被清理掉，

**效率问题：** 基于引用计数的回收机制，每次回收内存，都需要把所有对象的引用计数都遍历一遍，这是非常消耗时间的，于是引入了分代回收来提高回收效率，分代回收采用的是用“空间换时间”的策略。


**分代回收**

核心思想：在历经多次扫描的情况下，都没有被回收的变量，gc机制就会认为，该变量是常用变量，gc对其扫描的频率会降低

1、分代 分代指的是根据存活时间来为变量划分不同等级（也就是不同的代）

新定义的变量，放到新生代这个等级中，假设每隔1分钟扫描新生代一次，如果发现变量依然被引用，那么该对象的权重（权重本质就是个整数）加一，当变量的权重大于某个设定得值（假设为3），会将它移动到更高一级的青春代，青春代的gc扫描的频率低于新生代（扫描时间间隔更长），假设5分钟扫描青春代一次，这样每次gc需要扫描的变量的总个数就变少了，节省了扫描的总时间，接下来，青春代中的对象，也会以同样的方式被移动到老年代中。也就是等级（代）越高，被垃圾回收机制扫描的频率越低

2、回收 回收依然是使用引用计数作为回收的依据


**缺点：**

例如一个变量刚刚从新生代移入青春代，该变量的绑定关系就解除了，该变量应该被回收，但青春代的扫描频率低于新生代，这就到导致了应该被回收的垃圾没有得到及时地清理。

没有十全十美的方案： 毫无疑问，如果没有分代回收，即引用计数机制一直不停地对所有变量进行全体扫描，可以更及时地清理掉垃圾占用的内存，但这种一直不停地对所有变量进行全体扫描的方式效率极低，所以我们只能将二者中和。

综上： 垃圾回收机制是在清理垃圾&释放内存的大背景下，允许分代回收以极小部分垃圾不会被及时释放为代价，以此换取引用计数整体扫描频率的降低，从而提升其性能，这是一种以空间换时间的解决方案


**查看进程占用的内存**

```python
import resource

mem_init = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
l = []
for i in range(500000):
    l.append(object())
mem_final = resource.getrusage(resource.RUSAGE_SELF).ru_maxrss
del l
print('Class: {}:\n'.format(getattr(cls, '__name__')))
print('Initial RAM usage: {:14,}'.format(mem_init))
print('  Final RAM usage: {:14,}'.format(mem_final))
```

内存使用就是 mem_final - mem_init。但是要注意 2 点：

1. 不同平台上 ru_maxrss 的值的单位是不一样的，在 OS X 上单位是 Byte，但是在 Linux 上单位是 KB。我之前用惯了 OS X，一次查看现在程序内存使用，看到上述方法的返回值太小，数量级上差了好多，觉得明显不对啊，困惑了很久，最后还是直接去翻 [libbc 的手册](https://www.gnu.org/software/libc/manual/html_node/Resource-Usage.html) 才知道这个区别。大家要注意。
2. 上面用到的 resource.RUSAGE_SELF 表示当前进程自己，如果你希望知道该进程下已结束子进程的内存也计算进来，需要使用 resource.RUSAGE_CHILDREN。另外还有一个 RUSAGE_BOTH 相当于当前进程和子进程自己的总和，不过这个是平台相关的，你要先了解你是用的发行版本是否提供。

## 弱引用

基本的 list 和 dict 实 例不能作为所指对象，但是它们的子类可以。
int 和 tuple 实例不能作 为弱引用的目标，甚至它们的子类也不行。
```python
class MyList(list):
 """list的子类，实例可以作为弱引用的目标"""
a_list = MyList(range(10))
# a_list可以作为弱引用的目标
wref_to_a_list = weakref.ref(a_list)
```

```python
import threading
import weakref

class Data:
    def __init__(self, key):
        pass

# 数据缓存
class Cacher:
    def __init__(self):
        self.pool = {}
        self.lock = threading.Lock()
        
    def get(self, key):
        with self.lock:
            r = self.pool.get(key)
            if r:
                # 调用弱引用对象，即可找到指向的对象
                # 删除变量后，再次调用弱引用对象返回None
                data = r()
                if data:
                    print(f'cache hits {key}')
                    return data
            
            print(f'cache set {key}')
            data = Data(key)
            # 创建一个指向该数据的弱引用
            self.pool[key] = weakref.ref(data)  
            return data
            
def func(cacher):
    for i in range(5):
        data = cacher.get(i)
    
if __name__ == '__main__':
    cacher = Cacher()
    
    threads = []
    for i in range(3):
        thread = threading.Thread(target=func, args=(cacher,))
        thread.start()
        threads.append(thread)
        
    for thread in threads:
        thread.join()
```

- _weakref.WeakKeyDictionary_ ，键只保存弱引用的映射类（一旦键不再有强引用，键值对条目将自动消失）；
- _weakref.WeakValueDictionary_ ，值只保存弱引用的映射类（一旦值不再有强引用，键值对条目将自动消失）；

```python
```text
import threading
import weakref
# 数据缓存
class Cacher:
    def __init__(self):
        self.pool = weakref.WeakValueDictionary()
        self.lock = threading.Lock()
    def get(self, key):
        with self.lock:
            data = self.pool.get(key)
            if data:
                return data
            self.pool[key] = data = Data(key)
            return data
```
```

## 文件读取

```python
#1 读取文本文件
f = open('a.txt', 'r', encoding='utf-8') 

f.read()   # 读取文件全部内容,返回str
f.read(6)  # 每次最多读取6个字节的内容,返回str
f.readline()   # 每次读取一行内容,返回str
f.readlines()  # 读取文件全部内容,按行返回list

f.close()

#2 使用上下文管理器自动关闭文件
with open('test.txt', 'r') as f:
    for line in f:
        print(line)

#3 读取二进制文件
with open('test.bin', 'rb') as f:
    while True:
        # 每次读入1024个字节到内存中
        data=f.read(1024) 
        if len(data) == 0:
            break
        print(data)

```

# 模块和包

在Python中，一个py文件就是一个模块，文件名为xxx.py模块名则是xxx。

**如有两个文件：**
foo.py
```Python
x=1
def get():
    print(x)
def change():
    global x
    x=0
class Foo:
    def func(self):
       print('from the func')
```
main.py
```Python
import foo    #导入模块foo
a=foo.x       #引用模块foo中变量x的值赋值给当前名称空间中的名字a
foo.get()     #调用模块foo的get函数
foo.change()  #调用模块foo中的change函数
obj=foo.Foo() #使用模块foo的类Foo来实例化，进一步可以执行obj.func()
```
首次导入模块会做三件事：  
1、执行源文件代码  
2、产生一个新的名称空间用于存放源文件执行过程中产生的名字  
3、在当前执行文件所在的名称空间中得到一个名字foo，该名字指向新创建的模块名称空间，若要引用模块名称空间中的名字，需要加上该前缀

第一次导入模块已经将其加载到内存空间了，之后的重复导入会直接引用内存中已存在的模块，不会重复执行文件，通过import sys，打印sys.modules的值可以看到内存中已经加载的模块名。


**循环导入问题**
在一个模块加载导入的过程中导入另外一个模块，而在另外一个模块中又返回来导入第一个模块中的名字，由于第一个模块尚未加载完毕，所以引用失败、抛出异常，究其根源就是在python中，同一个模块只会在第一次导入时执行其内部代码，再次导入该模块时，即便是该模块尚未完全加载完毕也不会去重复执行内部代码

```Python
# m1.py
print('正在导入m1')
from m2 import y

x='m1'


# m2.py
print('正在导入m2')
from m1 import x

y='m2'


# run.py
import m1
```

[1.执行run.py](http://1.xn--run-m35fr68n.py)

```Python
正在导入m1
正在导入m2
Traceback (most recent call last):
  File "/Users/linhaifeng/PycharmProjects/pro01/1 aaaa练习目录/run.py", line 1, in <module>
    import m1
  File "/Users/linhaifeng/PycharmProjects/pro01/1 aaaa练习目录/m1.py", line 2, in <module>
    from m2 import y
  File "/Users/linhaifeng/PycharmProjects/pro01/1 aaaa练习目录/m2.py", line 2, in <module>
    from m1 import x
ImportError: cannot import name 'x'
```

先执行run.py--->执行import m1，开始导入m1并运行其内部代码--->打印内容"正在导入m1"  
--->执行from m2 import y 开始导入m2并运行其内部代码--->打印内容“正在导入m2”--->执行from m1 import x,由于m1已经被导入过了，所以不会重新导入，所以直接去m1中拿x，然而x此时并没有存在于m1中，所以报错

[2.执行m1.py](http://2.xn--m1-jn5dy99k.py)：执行文件不等于导入文件，比如执行m1.py不等于导入了m1

```Python
正在导入m1
正在导入m2
正在导入m1
Traceback (most recent call last):
  File "/Users/linhaifeng/PycharmProjects/pro01/1 aaaa练习目录/m1.py", line 2, in <module>
    from m2 import y
  File "/Users/linhaifeng/PycharmProjects/pro01/1 aaaa练习目录/m2.py", line 2, in <module>
    from m1 import x
  File "/Users/linhaifeng/PycharmProjects/pro01/1 aaaa练习目录/m1.py", line 2, in <module>
    from m2 import y
ImportError: cannot import name 'y'
```

[执行m1.py](http://xn--m1-jn5dy99k.py)，打印“正在导入m1”，执行from m2 import y ，导入m2进而执行m2.py内部代码--->打印"正在导入m2"，执行from m1 import x，此时m1是第一次被导入，执行m1.py并不等于导入了m1，于是开始导入m1并执行其内部代码--->打印"正在导入m1"，执行from m1 import y，由于m1已经被导入过了，所以无需继续导入而直接问m2要y，然而y此时并没有存在于m2中所以报错

```Python
# 解决方案1：导入语句放到最后，保证在导入时，所有名字都已经加载过

# m1.py
print('正在导入m1')

x='m1'

from m2 import y

# m2.py
print('正在导入m2')
y='m2'

from m1 import x

# run.py
import m1
print(m1.x)
print(m1.y)


# 解决方案2：导入语句放到函数中，只有在调用函数时才会执行其内部代码

# m1.py
print('正在导入m1')

def f1():
    from m2 import y
    print(x,y)

x = 'm1'

# m2.py
print('正在导入m2')

def f2():
    from m1 import x
    print(x,y)

y = 'm2'

# run.py
import m1

m1.f1()


```

**包**
包就是一个含有__init__.py文件的文件夹，文件夹内可以组织子模块或子包，导包就是在导包下__init__.py文件  
示例：

```Python
pool/                #顶级包
├── __init__.py     
├── futures          #子包
│   ├── __init__.py
│   ├── process.py
│   └── thread.py
└── versions.py      #子模块


# process.py
class ProcessPoolExecutor:
    def __init__(self,max_workers):
        self.max_workers=max_workers

    def submit(self):
        print('ProcessPool submit')

# thread.py
class ThreadPoolExecutor:
    def __init__(self, max_workers):
        self.max_workers = max_workers

    def submit(self):
        print('ThreadPool submit')

# versions.py
def check():
    print('check versions’)

# __init__.py文件内容均为空 
```

包属于模块的一种，因而包以及包内的模块均是用来被导入使用的，而绝非被直接执行，首次导入包（如import pool）同样会做三件事：

1、执行包下的init.py文件

2、产生一个新的名称空间用于存放init.py执行过程中产生的名字

3、在当前执行文件所在的名称空间中得到一个名字pool，该名字指向init.py的名称空间，例如pool.xxx和pool.yyy中的xxx和yyy都是来自于pool下的init.py，也就是说导入包时并不会导入包下所有的子模块与子包

```Python
import pool

pool.versions.check() #抛出异常AttributeError
pool.futures.process.ProcessPoolExecutor(3) #抛出异常AttributeError
```

绝对导入：以顶级包为起始

```Python
#pool下的__init__.py
from pool import versions
from pool import futures

#futrues下的__init__.py
from pool.futures import process
```

相对导入：.代表当前文件所在的目录，..代表当前目录的上一级目录，依此类推  
相对导入只能在包内部使用，用相对导入不同目录下的模块是非法的，而且...取上一级不能出包

```Python
#pool下的__init__.py
from . import versions
from . import futures

#futrues下的__init__.py
from . import process
```

通过操作init.py可以“隐藏”包内部的目录结构，降低使用难度，比如想要让使用者直接使用

```Python
import pool

pool.check()
pool.ProcessPoolExecutor(3)
pool.ThreadPoolExecutor(3)

# 需要操作pool下的__init__.py
from .versions import check
from .futures.process import ProcessPoolExecutor
from .futures.thread import ThreadPoolExecutor
```

延迟加载
`pip install apipkg`
```python
import importlib

module = importlib.import_module(module_name)
```
