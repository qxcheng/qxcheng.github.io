---
categories:
- - Python
date: 2024-04-16 13:52:47
description: 'Python基础6-单元测试'
id: '82'
tags:
- python
title: Python基础6-单元测试
---


## 1.unitest

```Python
# 单元测试
# test.py
import unittest  

class TestList(unittest.TestCase):
    # 测试方法以test开头
    def test_pop(self):
        l = [1, 2, 3]
        r = l.pop()
        self.assertEqual(l, [1,2])
        self.assertTrue(r == 3)
        self.assertFalse(r == 1)

    def test_index(self):
        l = [1, 2, 3]
        n = l.index(2)
        self.assertEqual(n, 1)
        # 断言会抛出指定类型的Error
        with self.assertRaises(ValueError):
            n = l.index(4)   

    # 每次调用一个测试方法前执行
    def setUp(self):
        print('\nsetUp...')

    # 每次调用一个测试方法后执行
    def tearDown(self):
        print('tearDown...')

if __name__ == '__main__':
    unittest.main()

$ python test.py -v 
test_index (__main__.TestList.test_index) ...
setUp...
tearDown...
ok
test_pop (__main__.TestList.test_pop) ...
setUp...
tearDown...
ok

----------------------------------------------------------------------
Ran 2 tests in 0.001s

OK
```

## 2.pytest

```python
# test_demo.py 
# 默认发现当前目录下 test_*.py 或 *_test.py
# 测试方法以test开头
# 测试类以Test开头

import pytest

def test_int():
    assert int('100') == 100

class TestClass:
    def test_str(self):
        assert str(100) == '100'

    def test_list(self):
        x = []
        with pytest.raises(IndexError):
            print(x[0])

$ pytest 
=========================== test session starts ===========================
platform win32 -- Python 3.11.4, pytest-7.4.0, pluggy-1.3.0
rootdir: C:\test
plugins: anyio-3.7.1
collected 3 items

main.py ...                                                          [100%]

============================ 3 passed in 0.02s ============================
```

### 测试夹具

如果将测试夹具写在`conftest.py` 文件中，则该包内所有模块均可使用该测试夹具。

```python
# test_demo.py
import pytest
import random

@pytest.fixture(scope="function")
def random_num():
    return random.randint(1, 10)

def test_1(random_num):
    assert random_num < 100

def test_2(random_num):        # random_num 取值不一样
    assert random_num > 100

# scope取值代表重新生成 fixture 的时机：
function: 调用测试函数前重新生成
class: 调用测试类前重新生成 
module: 载入测试模块前重新生成
package: 载入包前重新生成
session: 运行所有用例前，只生成一次
```

### 前置和清理

```python
import pytest

@pytest.fixture(scope="module")
def file():
    with open('test.txt', 'r') as f:
        print("before")
        yield f
        print("after")

def test_file(file):
    assert file.read() == 'HelloWorld\n'
```

### 跳过测试

```python
import pytest
import os

# 直接跳过
@pytest.mark.skip(reason="waiting for test")
def test_int(file):
    assert int('10') == 2

# 条件跳过
@pytest.mark.skipif(os.name == 'nt', reason="requires linux")
def test_str():
    assert str(10) == '2'

# 预计失败
@pytest.mark.xfail
def test_list():
    a = []
    assert a[0] == 0

PS C:\test> pytest
=========================== test session starts ===========================
platform win32 -- Python 3.11.4, pytest-7.4.0, pluggy-1.3.0
rootdir: C:\test
plugins: anyio-3.7.1
collected 3 items

test_demo.py ssx                                                     [100%]

====================== 2 skipped, 1 xfailed in 0.13s ======================
```

### 参数化测试

```python
import pytest

@pytest.mark.parametrize(
    "eval_str,expected",
    [("8", 8), ("list((1,2,3))",[1,2,3]),
     pytest.param("7*8", 64, marks=pytest.mark.xfail)],
)
def test_eval(eval_str, expected):
    assert eval(eval_str) == expected

PS D:\Projects\test> pytest
============================================ test session starts ============================================
platform win32 -- Python 3.11.4, pytest-7.4.0, pluggy-1.3.0
rootdir: D:\Projects\test
collected 3 items

test_demo.py ..x                                                                                       [100%]

======================================= 2 passed, 1 xfailed in 0.09s ========================================
```