---
categories:
- - Python
date: 2024-04-17 14:22:33
description: 'Python常用库10-C扩展'
id: '130'
tags:
- C语言
- python
title: Python常用库10-C扩展
---


## 1.swig

Simplified Wrapper and Interface Generator Python开发者一般不采用，因为会带来不必要的复杂。 当有一个C/C++代码库需要被多种语言调用时，这将是个不错的选择。

```Python
# test.c
'''
#include <time.h>
int fact(int n) 
{
    if (n <= 1) return 1;
    else return n*fact(n-1);
}
int my_mod(int x, int y) 
{
    return (x%y);
}
char *get_time()
{
    time_t ltime;
    time(&ltime);
    return ctime(&ltime);
}
'''

# test.h
'''
int fact(int n);
int my_mod(int x, int y);
char *get_time();
'''

# test.i
'''
%module test
%{
#include <time.h>
#include "test.h"
%}
%include "test.h"

//extern int fact(int n) # 不用头文件可直接这样写
'''

# 编译
swig -python test.i
gcc -c -fPIC test.c test_wrap.c -I/usr/include/python2.7
gcc -shared test.o test_wrap.o -o _test.so

# Python调用
import test
test.fact(5)     # 120
test.my_mod(7,3) # 1
test.get_time()  # 'Sun Feb 11 23:01:07 1996'
```

## 2.Python/C API

Python/C API可能是最广泛使用的。可在C代码中操作Python对象。

```C
//1.addList.c
#include <Python.h>   
#define PyInt_AsLong(x) (PyLong_AsLong((x))) 

//实现的函数，addList_add: 模块名_函数名（python中）
static PyObject* addList_add(PyObject* self, PyObject* args) 
{
    PyObject* listObj;

    //解析参数
    if (! PyArg_ParseTuple( args, "O", &listObj ))
        return NULL;
    /* 注：
    若传入一个字符串，一个整数和一个Python列表，则这样写:
    int n;
    char *s;
    PyObject* list;
    PyArg_ParseTuple(args, "siO", &s, &n, &list);
    */

    long length = PyList_Size(listObj);  //获取长度
    int i, sum =0;
    for (i = 0; i < length; i++)
    {
        PyObject* temp = PyList_GetItem(listObj, i); //获取每个元素
        long elem = PyInt_AsLong(temp);              //将PyInt对象转换为长整型
        sum += elem;
    }
    return Py_BuildValue("i", sum); //长整型sum被转化为Python整形对象并返回给Python代码
}

//实现的函数的信息表，每行一个函数，以空行作为结束
static PyMethodDef addList_funcs[] = 
{
    {"add", (PyCFunction)addList_add, METH_VARARGS, "Add all elements of the list."},
    {NULL, NULL, 0, NULL}
};

//模块定义结构
static struct PyModuleDef addList_module = {
    PyModuleDef_HEAD_INIT,
    "addList",   /* 模块名 */
    "",          /* 模块文档 */
    -1,          /* size of per-interpreter state of the module,
                 or -1 if the module keeps state in global variables. */
    addList_funcs
};

//模块初始化
PyMODINIT_FUNC PyInit_addList(void)
{
    return PyModule_Create(&addList_module);
}

//2 setup.py
from distutils.core import setup, Extension
setup(name='addList', version='1.0', ext_modules=[Extension('addList', ['addList.c'])])

//3 安装为Python模块
sudo python setup.py install

//4 在Python中调用
import addList
l = [1,2,3,4,5]
print(str(addList.add(l)))  # 15
```

## 3.ctypes

```C
ctypes允许调用C函数时使用Python中的字符串型和整型。
而其他类似布尔型和浮点型，必须转换。
C类型                       Python类型                        ctypes 类型
char                        1-character/string                c_char
wchar_t                     1-character/Unicode、string       c_wchar
char                        int/long                          c_byte
char                        int/long                          c_ubyte
short                       int/long                          c_short
unsigned short              int/long                          c_ushort
int                         int/long                          C_int
unsigned int                int/long                          c_uint
long                        int/long                          c_long
unsigned long               int/long                          c_ulong
long long                   int/long                          c_longlong
unsigned long long          int/long                          c_ulonglong
float                       float                             c_float
double                      float                             c_double
char *(NULL terminated)     string or none                    c_char_p
wchar_t *(NULL terminated)  unicode or none                   c_wchar_p
void *                      int/long or none                  c_void_p
POINTER(POINTER(POINTER(c_int)))
```

### 基本使用

```Python
//1 test.c
#include <stdio.h>

int add_int(int num1, int num2)
{
    return num1 + num2;
}

float add_float(float num1, float num2)
{
    return num1 + num2;
}

char* str_print(char *str)  
{  
    puts(str);  
    return str;  
} 

float add_float_list(float* num, int length)
{
    float sum=0;
    for(int i=0; i<length; i++)
    {
        sum += num[i];
    }
    return sum;
}

void str_list_print(char** str_list, int length)
{
    for(int i=0; i<length; i++)
    {
        puts(str_list[i]);
    }
}

// 将test.c编译为.so文件(windows下为DLL)
gcc -fPIC -shared -Wl,-soname,test test.c -o test.so   
gcc -fPIC -shared test.c -o test.so 

#2 test.py
from ctypes import *

lib = CDLL('./test.so')   # 加载.so动态库

# 传整数
res_int = lib.add_int(4,5)
print("Sum of 4 and 5 = " + str(res_int))

# 传浮点数
a = c_float(5.5)                 # 浮点数需要先进行转换
b = c_float(4.1)
lib.add_float.restype = c_float  # 返回值类型转换
print("Sum of 5.5 and 4.1 = ", str(lib.add_float(a, b)))

# 传字符串
lib.str_print.restype = c_char_p
res_str = lib.str_print(b'Hello')
print(res_str)

# 传浮点数列表
c = (c_float*5)()
for i in range(len(c)):
    c[i] = i
lib.add_float_list.restype = c_float
print("Sum of list = ", str(lib.add_float_list(c, len(c))))

# 传字符串列表
d = (c_char_p*3)()
for i in range(len(d)):
    d[i] = b'HELLO'
lib.str_list_print(d, len(d))
```

### 指针使用

```Python
1. test.c
#include <stdio.h>

void one_ptr_func(int* num, int length)
{
    for(int i=0; i<length; i++)
    {
        num[i] *= 2;
    }
}

void two_ptr_func(int** num, int row, int column)
{
    for(int i=0; i<row; i++)
    {
        for(int j=0; j<column; j++)
        {
            num[i][j] *= 2;
        }    
    }
}

void three_ptr_func(int*** num, int x, int row, int column)
{
    for(int a=0; a<x; a++){
        for(int i=0; i<row; i++){
            for(int j=0; j<column; j++){     
                num[a][i][j] *= 2;
            }    
        }
    }
}

2. test.py
from ctypes import *

lib = CDLL('./test.so')   # 加载.so动态库

# 一级指针
data = [1,2,3,4,5]
one_arr = (c_int*5)(*data)              #一维数组
one_ptr = cast(one_arr, POINTER(c_int)) #一维数组转换为一级指针

lib.one_ptr_func(one_ptr, 5)
for i in range(5):
    print(one_ptr[i], end=' ')
print("\n")

# 二级指针
data = [(1,2,3), (4,5,6)]
two_arr = (c_int*3*2)(*data)                  #二维数组
one_ptr_list = [] 
for i in range(2):
    one_ptr_list.append(cast(two_arr[i], POINTER(c_int))) #一级指针添加进列表
two_ptr=(POINTER(c_int)*2)(*one_ptr_list)                 #转化为二级指针

lib.two_ptr_func(two_ptr, 2, 3)
for i in range(2):
    for j in range(3):
        print(two_ptr[i][j], end=' ')
    print("\n")

# 三级指针
data =[((1,2,3), (4,5,6)),((7,8,9), (10,11,12))]      #2x2x3
three_arr =(c_int*3*2*2)(*data)                       #三维数组
two_ptr=[]
for i in range(2):
    one_ptr=[]
    for j in range(2):
        one_ptr.append(cast(three_arr[i][j], POINTER(c_int))) #一级指针添加进列表   
    two_ptr.append((POINTER(c_int)*2)(*one_ptr))              #转化为二级指针添加进列表                     
three_ptr = (POINTER(POINTER(c_int))*2)(*two_ptr)             #转换为三级指针

lib.three_ptr_func(three_ptr, 2, 2, 3)
for i in range(2):
    for j in range(2):
        for k in range(3):
            print(three_ptr[i][j][k], end=' ')
        print("\n")
```

### numpy使用

```Python
1. test.c
如上

2. test.py
from ctypes import *
import numpy as np

lib = CDLL('./test.so')   # 加载.so动态库

a = np.asarray(range(16), dtype=np.int32)
if not a.flags['C_CONTIGUOUS']:
    a = np.ascontiguous(a, dtype=data.dtype)   # 如果不是C连续的内存，必须强制转换
ptr = cast(a.ctypes.data, POINTER(c_int))      # 转换为一级指针
for i in range(16):
    print(ptr[i], end=' ')
print('\n')

lib.one_ptr_func(ptr, 16)   
for i in range(16):
    print(a[i], end=' ')  # 注意此时变量a也被改变了
print('\n')

3. 方法2：使用numpy.ctypeslib
void np_ptr_func(int* num, int x, int row, int column)
{
    for(int a=0; a<x; a++){
        for(int i=0; i<row; i++){
            for(int j=0; j<column; j++){     
                num[a*row*column+i*column+j] *= 2;
            }    
        }
    }
} 

from ctypes import *
import numpy as np
import numpy.ctypeslib as npct

a = np.ones((3, 3, 3), np.int32)
lib = npct.load_library("test", ".")                 #引入动态链接库
lib.np_ptr_func.argtypes = [npct.ndpointer(dtype=np.int32, ndim=3, flags="C_CONTIGUOUS"),
                            c_int, c_int, c_int]     #参数说明
lib.np_ptr_func(a, c_int(3), c_int(3), c_int(3))     #函数调用

for i in range(3):
    for j in range(3):
        for k in range(3):
            print(a[i][j][k], end=' ')  
        print('\n')
```