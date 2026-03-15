---
categories:
- - Python
date: 2024-04-16 10:51:14
description: 'Python基础1-数据结构'
id: '50'
tags:
- python
title: Python基础1-数据结构
---


## 1.字符串

### 大小写

```python
# 全部大写
print('Hello'.upper())      # HELLO

# 全部小写
print('Hello'.lower())      # hello

# 大小写互换
print('Hello'.swapcase())   # hELLO     

# 首字母大写，其余小写
print('hELLO'.capitalize()) # Hello
```

### 去空格

```python
# 去两边空格
print(' Hello '.strip())    # 'Hello'

# 去左边空格        
print(' Hello '.lstrip())   # 'Hello '        

# 去右边空格
print(' Hello '.rstrip())   # ' Hello' 

# 去两边所有指定的字符       
print('Hello'.strip('Ho'))  # 'ell'        

# 去左边所有指定的字符
print('Hello'.lstrip('lo')) # 'Hello'    

# 去右边所有指定的字符    
print('Hello'.rstrip('lo')) # 'He'
```

### 对齐

```python
# 固定长度右对齐，左边不够用空格补齐
print('Hello'.rjust(7))  # '  Hello'

# 固定长度左对齐，右边不够用空格补齐
print('Hello'.ljust(7))  # 'Hello  '

# 固定长度居中对齐，两边不够用空格补齐   
print('Hello'.center(7)) # ' Hello ' 

# 固定长度右对齐，左边不够用0补齐 
print('Hello'.zfill(7))  # '00Hello'
```

### 判断

```python
# 是否以'He'开头,可传元组
print('Hello'.startswith('He')) # True     

# 是否以'lo'结尾
print('Hello'.endswith('lo'))   # True  

# 是否全为字母或数字
print('Hello123'.isalnum())     # True

# 是否全字母       
print('Hello'.isalpha())        # True

# 是否全数字     
print('Hello123'.isdigit())     # False  

# 是否全小写 
print('Hello'.islower())        # False

# 是否全大写       
print('Hello'.isupper())        # False
```

### 查找

```python
# 搜索字符串第一次出现的位置，没有返回-1
print('Hello'.find('o'))     # 4

# 从指定起始位置开始搜索         
print('Hello'.find('o',1))   # 4    

# 指定起始及结束位置搜索  
print('Hello'.find('o',1,3)) # -1  

# 从右边开始查找
print('Hello'.rfind('o'))    # 4

# 搜索到多少个指定字符串          
print('Hello'.count('o'))    # 1
```

### 替换

```python
# 替换所有'l'为'm'
print('Hello'.replace('l', 'm'))    
# Hemmo 

# 替换指定次数的'l'为'm'
print('Hello'.replace('l', 'm', 1)) 
# Hemlo 

# 每个tab替换为4个空格，默认为8个
print('    Hello'.expandtabs(4))       
# '    Hello'
```

### 分割与连接

```python
# 使用空格分割字符串，返回列表
print('H e l l o'.split(' ')) 
# ['H', 'e', 'l', 'l', 'o']           

# 按行分割符分为一个列表，True表示保留行分割符
print('Hello\\\\nWorld'.splitlines(True)) 
# ['Hello\\\\n', 'World']

# 把seq序列用''连接    
print(''.join(['H','e','l','l','o']))  
# Hello
```

### 映射

```python
# 将abcde映射到ABCDE
table = str.maketrans('abcde', 'ABCDE')
'abc'.translate(table)
# 'ABC'
```

## 2.列表

### 列表创建

```python
l = []                      
print(l)  # []

l = list()                  
print(l)  # []

l = [1,2,3] + [4,5,6]       
print(l)  # [1,2,3,4,5,6]

l = ['Hi'] * 3              
print(l)  # ['Hi','Hi','Hi']
```

### 列表切片

```python
l = [1,2,3,4,5,6]

print(l[0:3])     # [1, 2, 3]

print(l[:3])      # [1, 2, 3]

print(l[-3:-1])   # [4, 5]

print(l[-3:])     # [4, 5, 6]

print(l[0:10:2])  # [1, 3, 5]

print(l[:])       # [1, 2, 3, 4, 5, 6]

print(l[::-1])    # [6, 5, 4, 3, 2, 1]
```

### 列表操作

```python
l = [1,2,3]

l.append(4)     # 末尾追加元素      
print(l)        # [1, 2, 3, 4]  

l.insert(0, -1) # 将元素插入指定位置
print(l)        # [-1, 1, 2, 3, 4]

r = l.pop()     # 移除末尾元素并返回
print(l, r)     # [-1, 1, 2, 3] 4 

r = l.pop(0)    # 移除指定位置元素并返回
print(l, r)     # [1, 2, 3] -1 

l.remove(1)     # 移除指定元素的第一个匹配项
print(l)        # [2, 3]  

l.extend([4,5]) # 用新列表扩展原列表
print(l)        # [2, 3, 4, 5] 

print(l.count(2)) # 1 统计元素出现次数
print(l.index(2)) # 0 返回第一个匹配项的索引

l.reverse()       # 反向列表中元素
print(l)          # [5, 4, 3, 2]

l.sort()          # 对列表进行排序
print(l)          # [2, 3, 4, 5]

t = (1,2,3)       # 将元组转换为列表
print(list(t))    # [1, 2, 3]
```

### 列表表达式

```python
print([i*i for i in range(10) if i%2])
# [1, 9, 25, 49, 81]

print([m+n for m in 'AB' for n in 'XY'])
# ['AX', 'AY', 'BX', 'BY']
```

## 3.元组

```python
# 元组初始化后不能修改

t = ()
print(t)  # ()

t = (50,)
print(t)  # (50,)

t = ('A', 'B', 1, 2)
print(t)  # ('A', 'B', 1, 2)

# 将列表转换为元组
print(tuple([1,2]))          # (1, 2)

# 传入字典返回key的元组       
print(tuple({1:'A', 2:'B'})) # (1, 2)
```

## 4.集合

```python
# 1.集合是无序的和无重复元素的
# 2.集合中不能放入可变对象

s = {'A', 'B'} 
print(s)       # {'B', 'A'}

s.add('CD')    # 添加整体
print(s)       # {'B', 'A', 'CD'}

s.update('CD') # 添加每个元素
print(s)       # {'B', 'C', 'D', 'A', 'CD'}

s.remove('CD') # 移除元素
print(s)       # {'B', 'C', 'D', 'A'}

s2 = {'A', 'D'}
print(s & s2)  # 交集 {'D', 'A'}
print(s  s2)  # 并集 {'B', 'C', 'D', 'A'}
print(s - s2)  # 差集 {'B', 'C'}

# 集合推导式
s = {x*x for x in range(5)}
print(s)       # {0, 1, 4, 9, 16}
```

## 5.字典

```python
d = {1:'A', 2:'B'} # key为不可变对象

# 返回所有键值对
print(d.items())   # [(1, 'A'), (2, 'B')]

# 返回所有键
print(d.keys())    # [1, 2]  

# 返回所有值
print(d.values())  # ['A', 'B']

# 判断键是否存在
print(1 in d)      # True

# 返回指定键的值，键不存在返回None
print(d.get(3, None)) # None        

# 返回指定键的值，键不存在添加键并赋值为None
print(d.setdefault(3, None)) # None 
print(d)   # {1: 'A', 2: 'B', 3: None}

d.update({3:'C'}) # 添加键值对
print(d)   # {1: 'A', 2: 'B', 3: 'C'}            

d.pop(3)   # 根据指定的键删除键值对
print(d)   # {1: 'A', 2: 'B'}

del d[2]   # 删除键是2的键值对
print(d)   # {1: 'A'}

d.clear()  # 清空字典
print(d)   # {}

# 字典推导式
d = {1:'A', 2:'B'}
d2 = {key: value for key, value in d.items() if key > 1}
print(d2) # {2: 'B'}

# 快速对换字典的键和值
d3 = {value: key for key, value in  d.items()} 
print(d3) # {'A': 1, 'B': 2}
```

## 6.枚举

```python
from enum import Enum, unique

#1 直接使用枚举类
Week = Enum('Week', ('Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'))
print(Week.Mon.name)  # Mon
print(Week.Mon.value) # 1

for _, week in Week.__members__.items():
    print(week.name, week.value) 
    # Mon 1
    # Tue 2
    # ...

#2 自定义继承枚举类
@unique  # 保证没有重复值
class Weekday(Enum):
    Sun = 0 
    Mon = 1
    Tue = 2
    Wed = 3
    Thu = 4
    Fri = 5
    Sat = 6

print(Weekday.Sun.name)   # Sun
print(Weekday.Sun.value)  # 0
```

## 7.队列

### 先进先出队列

```python
# FIFO先进先出队列
from queue import Queue

# maxsize设置队列中数据上限，小于或等于0则不限制，队列中数据大于这个数则阻塞，直到队列中的数据被取出
q = Queue(maxsize=0)

# 往队列写入数据
q.put(0)
q.put(1)
q.put(2)

# 输出队列所有数据
print(q.queue)    # deque([0, 1, 2])

# 取队头数据
r = q.get()
print(r, q.queue) # 0 deque([1, 2])
```

### 后进先出队列

```python
# LIFO后进先出队列（栈）
from queue import LifoQueue

lq = LifoQueue(maxsize=0)

# 往队列写入数据
lq.put(0)
lq.put(1)
lq.put(2)

# 输出队列所有数据
print(lq.queue)    # [0, 1, 2]

# 取队尾数据
r = lq.get()
print(r, lq.queue) # 2 [0, 1]
```

### 优先级队列

```python
# 优先级队列，优先级设置数越小等级越高
from queue import PriorityQueue

pq = PriorityQueue(maxsize=0)

# 往队列写入数据，设置优先级
pq.put((2,'a'))
pq.put((1,'c'))
pq.put((3,'b'))

# 输出队列全部数据
print(pq.queue)   
# [(1, 'c'), (2, 'a'), (3, 'b')]

# 按优先级取队列数据
r = pq.get()
print(r, pq.queue) 
# (1, 'c') [(2, 'a'), (3, 'b')]
```

### 双边队列

```python
# 双边队列
from queue import deque

dq = deque(['b','c'])

dq.append('d')     # 添加数据到队尾
dq.appendleft('a') # 添加数据到队左

# 输出队列所有数据
print(dq)           
# deque(['a', 'b', 'c', 'd'])

# 移除队尾元素并返回
print(dq.pop())     # d

# 移除队左元素并返回
print(dq.popleft()) # a
```

### 生产者消费者模型

```python
from queue import Queue
import threading
import time

q = Queue(maxsize=10)
count = 1

def produce(name):
    global count 
    while count<10:
        q.put('第{}个面包'.format(count))
        print('{}生产了第{}个面包\n'.format(name, count))
        count += 1
        time.sleep(0.3)

def cousume(name):
    global count 
    while count<10:
        print('{}消费掉{}\n'.format(name, q.get()))
        time.sleep(0.3)
        q.task_done()

#开启线程
p1 = threading.Thread(target=produce,args=('P1',))
c1 = threading.Thread(target=cousume,args=('C1',))
c2 = threading.Thread(target=cousume,args=('C2',))

p1.start()
c1.start()
c2.start()
```

## 8.collections

### 双向列表

```python
# 高效插入和删除的双向列表
from collections import deque

d = deque(['b', 'c'], maxlen=3) 
print(d) 
# deque(['b', 'c'], maxlen=3)

d.appendleft('a')
print(d) 
# deque(['a', 'b', 'c'], maxlen=3)

d.append('d')
print(d) 
# deque(['b', 'c', 'd'], maxlen=3)

d.extend(['e', 'f'])
print(d) 
# deque(['d', 'e', 'f'], maxlen=3)

d.extendleft(['c','b'])
print(d) 
# deque(['b', 'c', 'd'], maxlen=3)

d.pop()
print(d) 
# deque(['b', 'c'], maxlen=3)

d.popleft()
print(d) 
# deque(['c'], maxlen=3)
```

### 命名元组

```python
from collections import namedtuple

Point = namedtuple('Point', ['x', 'y'])
p = Point(1, 2)
print(p.x) # 1
print(p.y) # 2

p1 = p._replace(y=3) # 修改元素
print(p1)            # Point(x=1, y=3)

d = {'x':4, 'y':5}
p2 = p._replace(**d) # 修改元素
print(p2)            # Point(x=4, y=5)

p3 = p._asdict()     # 转换为字典 
print(p3)            
# OrderedDict([('x', 1), ('y', 2)])
```

### 默认值字典

```python
from collections import defaultdict

# 键值默认值设为'N/A'
dd = defaultdict(lambda: 'N/A')  
print(dd['key']) # N/A

# 键值默认值设为list
t = ((0, 'A'),(1, 'B'),(1, 'C'),)
dd = defaultdict(list)
for num, letter in t:
    dd[num].append(letter)
print(dd)
# defaultdict(<class 'list'>, {0: ['A'], 1: ['B', 'C']})
print(dd[1])
# ['B', 'C']
```

### 顺序字典

```python
from collections import OrderedDict

# OrderedDict的Key按插入顺序排列
od = OrderedDict([('c', 3), ('a', 1), ('b', 2)])  
print(od) 
# OrderedDict([('c', 3), ('a', 1), ('b', 2)])
```

### 计数器

```python
from collections import Counter

# 统计字符出现的个数
c = Counter()
for ch in 'aabc':
    c[ch] = c[ch] + 1
print(c) 
# Counter({'a': 2, 'b': 1, 'c': 1})

# 统计序列中的元素
t = ((0, 'A'),(1, 'B'),(1, 'C'),)
c = Counter(num for num, letter in t)
print(c) 
# Counter({1: 2, 0: 1})

# 统计文件
with open('test.txt', 'r') as f:
    line_count = Counter(f)
print(line_count)
```