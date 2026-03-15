---
categories:
- - Python
date: 2024-04-16 13:58:32
description: 'Python基础7-文件处理'
id: '84'
tags:
- python
title: Python基础7-文件处理
---


## 1.文件读取

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

## 2.文件写入

```Python
#1 写入文本文件
with open('test1.txt', 'w') as f:
    f.write('Hello World!')

#2 将print输出重定向到文件    
with open('test2.txt', 'wt') as f:
    print('Hello World!', file=f)

#3 文件追加
with open('test3.txt.txt', 'a') as f:
     f.write('Hello World!')

#4 写入二进制文件
with open('text3.bin', 'wb') as f:
    text = 'Hello World'
    f.write(text.encode('utf-8'))   
```

## 3.文件指针

```python
test.txt文件中内容为：HelloWorld

with open('test.txt', 'r', encoding='utf-8') as f:
    f.seek(5, 0)     # 从文件开头向后移动5个字节
    print(f.tell())  # 5
    print(f.read())  # World

with open('test.txt', 'rb') as f:
    f.seek(2, 1)     # 从当前位置往后移动2个字节
    f.seek(3, 1)     # 从当前位置往后移动3个字节
    print(f.read())  # b'World'

    f.seek(-5, 2)    # 从文件末尾向前移动5个字节
    print(f.read())  # b'World'
```

## 4.文件编码

```python
#3 文件编码
import sys

sys.getdefaultencoding()              #默认系统编码
encoding = 'latin-1'                  #读取未知编码时使用
with open('somefile.txt', 'rt') as f: #读写不同编码的文本数据，如ascii, latin-1, utf-8，utf-16
f = open('sample.txt', 'rt', encoding='ascii', errors='replace') #编码错误处理
g = open('sample.txt', 'rt', encoding='ascii', errors='ignore')

#4 换行符
#默认读取文本时将普通换行符转换为\n，输出时将\n转换为系统默认的换行符
Unix：\n
Windows：\r\n
#指定换行符
with open('somefile.txt', 'rt', newline='') as f:
```

## 5.os

```python
import os 

print(os.name)    # 操作系统类型 posix->linux\unix\macos nt->windows
print(os.environ) # 操作系统中定义的环境变量

print(os.getcwd())         # 当前工作目录
print(os.listdir('.'))     # 列出目录的文件
print(os.stat('test.txt')) # 获取元数据

os.rename('test.txt', 'test.py') # 重命名 
os.remove('test.py')             # 删除文件

os.mkdir('test')         # 创建目录 
os.chdir('test')         # 切换目录
os.rmdir('test')         # 删除目录
os.makedirs('dir1/dir2', exist_ok=True) # 创建多级目录，已存在时不报错

# 递归遍历文件夹
for root, dirs, files in os.walk('.'):
    for name in files:
        print(os.path.join(root, name))
    for name in dirs:
        print(os.path.join(root, name))

#获取所有文件
names = [name for name in os.listdir('.')
        if os.path.isfile(os.path.join('.', name))]
print(names)

#获取所有目录
dirnames = [name for name in os.listdir('.')
        if os.path.isdir(os.path.join('.', name))]
print(dirnames)

#获取txt文件
txtfiles = [name for name in os.listdir('.')
            if name.endswith('.txt')]
print(txtfiles)
```

## 6.os.path

```Python
print(os.path.abspath('.'))  # 查看绝对路径
print(os.path.exists('.'))   # 判断文件或目录是否存在
print(os.path.isdir('.'))    # 判断是否是目录
print(os.path.isfile('.'))   # 判断是否是文件
print(os.path.islink('.'))   # 判断是否是链接
print(os.path.realpath('.')) # 返回被链接的原始文件
print(os.path.getsize('test.txt'))  # 获取文件大小
print(os.path.getmtime('test.txt')) # 获取修改日期

path = '/home/test/test.txt'
print(os.path.basename(path)) #'test.txt' 获取文件名
print(os.path.dirname(path))  #'/home/test' 获取目录
print(os.path.split(path))    #('/home/test', 'test.txt') 目录分离 
print(os.path.splitext(path)) #('/home/test/test', '.txt') 目录分离

print(os.path.join('test', 'test.txt'))      # test/test.txt 目录合并
print(os.path.expanduser('~/test/test.txt')) # 展开绝对路径 
```

## 7.pathlib

```python
from pathlib import Path

p = Path.cwd()   # 当前目录
print(p.parent)  # 父目录

# 获取目录下所有文件和文件夹的路径
for file in p.iterdir():  
    print(file)

# 获取目录下符合条件文件的路径
for file in p.glob('*.txt'):
    print(file)

# 获取目录和子目录下符合条件文件的路径
for file in p.rglob('*.txt'):
    print(file)

# 创建多层文件夹
p = Path("./test/test")
if not p.exists():
    p.mkdir(parents=True)

p = Path("test.txt")
print(p.stat())       # 文件信息
print(p.is_dir())     # False
print(p.is_file())    # True
print(p.name)         # test.txt
print(p.suffix)       # .txt
p.rename("test1.txt") # 重命名
```

## 8.shutil

```python
import shutil

# 复制文件到目标文件夹
src = "test.txt"
dst = "./test"
shutil.copy(src, dst)           
shutil.copy(src, "./notexist")  # 目标文件夹不存在时，复制为文件notexist   

# 复制文件夹，dst需为空文件夹，dst不存在时自动创建
src = "./test"
dst = "./test1" 
shutil.copytree(src, dst)

# 移动文件或文件夹，dst不存在时自动创建
shutil.move(src, "./test2" )

# 删除文件夹
shutil.rmtree("./test2")
```

## 9.glob

```python
import glob
# 当前目录下存在：test.txt test1.txt

listglob = glob.glob(r"*.txt")          
print(listglob)  # ['test.txt', 'test1.txt']

listglob = glob.glob(r"test?.txt")
print(listglob)  # ['test1.txt']

listglob = glob.glob(r"test[0,1,2].txt")
print(listglob)  # ['test1.txt']

listglob = glob.glob(r"test[0-3].txt")
print(listglob)  # ['test1.txt']

listglob = glob.iglob(r"test[a-z].txt")   
print(listglob)  # <generator object _iglob at 0x0000022C4B4A9BC0>
```

## 10.zip

```python
import os
import zipfile

txtfiles = [name for name in os.listdir('.') if name.endswith('.txt')]

# 打包
with zipfile.ZipFile(r"test.zip", "w") as zipobj:
    for file in txtfiles:
        zipobj.write(file)

# 读包
with zipfile.ZipFile(r"test.zip", "r") as zipobj:
     print(zipobj.namelist())

# 解包单个文件
dst = './test'
with zipfile.ZipFile("test.zip", "r") as zipobj:
    zipobj.extract("test.txt", dst)

# 解包所有文件
dst = './test1'
with zipfile.ZipFile("test.zip", "r") as zipobj:
    zipobj.extractall(dst)
```