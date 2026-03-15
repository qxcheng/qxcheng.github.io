---
categories:
- - Python
date: 2024-04-17 13:15:51
description: 'Python常用库1-序列化、正则、哈希、base64'
id: '111'
tags:
- python
title: Python常用库1-序列化、正则、哈希、base64
---


## 1.序列化

### pickle

```Python
#1 pickle 
import pickle

d1  = dict(name='Tom', age=20, score=88)
s  = pickle.dumps(d1)   # 字典序列化为bytes

d2 = pickle.loads(s)    # bytes反序列化为字典
print(d2)  # {'name': 'Tom', 'age': 20, 'score': 88}

#2 将对象保存到文件中
with open('dump.txt', 'wb') as f:
    pickle.dump(d1, f)   

# 从文件中加载对象 ##加载时自动构造实例，注意安全
with open('dump.txt', 'rb') as f:
    d3 = pickle.load(f)  
    print(d3) # {'name': 'Tom', 'age': 20, 'score': 88}
```

### json

```Python
import json

#1 字典
# 字典序列化为json字符串
d = dict(name='Tom', age=20, score=80)
json_str = json.dumps(d)  
print(json_str)
# {"name": "Tom", "age": 20, "score": 80}

# json字符串反序列化为字典
json_str = '{"age": 20, "score": 80, "name": "Tom"}'
d = json.loads(json_str)  
print(d)
# {'age': 20, 'score': 80, 'name': 'Tom'}

#2 对象实例的序列化和反序列化
class Student(object):
    def __init__(self, name, age, score):
        self.name = name
        self.age = age
        self.score = score

def student2dict(std):
    return {
        'name': std.name,
        'age': std.age,
        'score': std.score
    }

def dict2student(d):
    return Student(d['name'], d['age'], d['score'])

s = Student('Tom', 20, 80)
json_str = json.dumps(s, default=student2dict)
print(json_str)
# {"name": "Tom", "age": 20, "score": 80}

json_str = '{"age": 20, "score": 80, "name": "Tom"}'
s = json.loads(json_str, object_hook=dict2student)
print(s.name, s.age, s.score)
# Tom 20 80

#3 文件
# json文件读
with open("../config/record.json",'r') as load_f:
    load_dict = json.load(load_f)

# json文件写
with open("../config/record.json","w") as dump_f:
    json.dump(load_dict,dump_f)

with open("../config/record.json", "w", encoding='utf-8') as dump_f:
    json.dump(result, dump_f, ensure_ascii=False)
```

## 2.re

```python
re.I 忽略大小写
re.L 表示特殊字符集 \w, \W, \b, \B, \s, \S 依赖于当前环境
re.M 多行模式
re.S 即为 . 并且包括换行符在内的任意字符（. 不包括换行符）
re.U 表示特殊字符集 \w, \W, \b, \B, \d, \D, \s, \S 依赖于 Unicode 字符属性数据库
re.X 为了增加可读性，忽略空格和 # 后面的注释

import re

string = 'Hello123World456Hello'

# 从起始位置匹配第一个
print(re.match('Hello', string).span()) # (0, 5)      
print(re.match('World', string))        # None

# 在整个字符串匹配第一个
print(re.search('Hello', string).span()) # (0, 5)      
print(re.search('World', string).span()) # (8, 13)

result = re.search(r'([A-Za-z]+)(\d+)', string)
print(result.group(0)) # Hello123
print(result.group(1)) # Hello
print(result.group(2)) # 123

# 匹配所有
pattern = re.compile(r'\d+')   
result = pattern.findall(string)
print(result) # ['123', '456']

pattern = re.compile(r'([A-Za-z]+)(\d+)')   
result = pattern.findall(string)
print(result) # [('Hello', '123'), ('World', '456')]

# 将匹配的子串替换
result = re.sub(r'[A-Za-z]+', '', string)
print(result) # 123456

# 将匹配的数字乘以2
def double(matched):
    value = int(matched.group('value'))
    return str(value * 2)
print(re.sub('(?P<value>\d+)', double, string))
# Hello246World912Hello

# 按照匹配的子串分割
result = re.split(r'[A-Za-z]+', string)
print(result) # ['', '123', '456', '']
```

## 3.哈希

```Python
import hashlib

# md5
md5 = hashlib.md5()
md5.update('hello'.encode('utf-8'))
md5.update('world'.encode('utf-8'))
print(md5.hexdigest())   # fc5e038d38a57032085441e7fe7010b0

# sha1
sha1 = hashlib.sha1()
sha1.update('hello'.encode('utf-8'))
sha1.update('world'.encode('utf-8'))
print(sha1.hexdigest())  # 6adfb183a4a2c94a2f92dab5ade762a47889a5a1

# 使用hmac实现带key的哈希
import hmac
message = b'Hello, world!'
key = b'secret'
h = hmac.new(key, message, digestmod='MD5') 
#如果消息很长，可以多次调用h.update(msg)
h.hexdigest()  #'fa4ee7d173f2d97ee79022d1a7355bcf'
```

## 4.base64

base64是一种用64个字符来表示二进制数据的方法。

```Python
import base64

# 二进制转base64
b64 = base64.b64encode(b'hello world')     
print(b64)     # b'aGVsbG8gd29ybGQ='

# base64转二进制
binary = base64.b64decode(b64) 
print(binary)  # b'hello world'

#处理URL时+/替换为-_
base64.b64encode(b'i\xb7\x1d\xfb\xef\xff')         #b'abcd++//'
base64.urlsafe_b64encode(b'i\xb7\x1d\xfb\xef\xff') #b'abcd--__' #把字符+和/分别变成-和_
base64.urlsafe_b64decode('abcd--__')               #b'i\xb7\x1d\xfb\xef\xff'
```