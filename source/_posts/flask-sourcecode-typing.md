---
categories:
- - flask
date: 2024-04-19 10:01:34
description: 本篇文章介绍flask框架中对typing模块的使用，vscode需要在软件右下角状态栏开启类型检查，其他ide请自行搜索启用方式。 
id: '163'
tags:
- python
- web框架
title: Flask源码系列2-typing模块
---


本篇文章介绍flask框架中对typing模块的使用，vscode需要在软件右下角状态栏开启类型检查，其他ide请自行搜索启用方式。 ![](https://www.bplan.top/2024-04-20231205131456.webp)

## 1.TYPE\_CHECKING

`typing.TYPE_CHECKING` 是在类型提示中用于解决循环导入问题的常量。在模块中引入 `TYPE_CHECKING` 时，可以使用类型提示而不会导致循环依赖。

flask源码：

```python
import typing as t

if t.TYPE_CHECKING:  # pragma: no cover
    from .testing import FlaskClient
    from .testing import FlaskCliRunner
```

示例：

```python
# module_a.py
from typing import List, TYPE_CHECKING

if TYPE_CHECKING:
    from module_b import SomeClass  # This import is only for type checking

class MyClass:
    def __init__(self, items: List['SomeClass']):
        self.items = items
```

```python
# module_b.py
from module_a import MyClass

class SomeClass:
    def __init__(self, value: str):
        self.value = value
```

在上述示例中，`module_a` 引入 `TYPE_CHECKING` 并使用它来导入 `SomeClass` 仅供类型检查使用，在实际运行时，`TYPE_CHECKING` 被设置为 `False`，因此实际运行时不会执行导入。

## 2.TypeVar

flask源码：

```python
# app.py
import typing as t
from . import typing as ft

T_teardown = t.TypeVar("T_teardown", bound=ft.TeardownCallable)

# typing.py
import typing as t

TeardownCallable = t.Union[
    t.Callable[[t.Optional[BaseException]], None],
    t.Callable[[t.Optional[BaseException]], t.Awaitable[None]],
]
```

*   `typing.TypeVar` 用于创建范型变量，`bound` 参数允许你约束范型变量的类型，使其必须是某个特定类型或其子类型。
*   `typing.Callable` 表示可调用对象，参数1为用列表表示的可调用对象的所有参数的类型，参数2为可调用对象的返回类型
*   `typing.Awaitable` 表示可等待对象。可等待对象通常是协程或类似于协程的对象，可以通过 `await` 关键字进行等待。

TypeVar示例： ![](https://www.bplan.top/2024-04-20231205130721.webp) 在这个例子中，`T` 必须是 `int` 类型或 `int` 的子类型。因此，`process_list` 函数期望接收一个包含整数的列表。如果你尝试传递一个包含其他类型的列表，类型检查器将会报错。

Awaitable示例：

```python
import asyncio
from typing import Awaitable

async def async_function() -> int:
    return 42

def regular_function() -> Awaitable[int]:
    # 返回一个 Awaitable 对象
    return async_function()

async def main():
    result = await regular_function()
    print(result)

# 运行异步程序
asyncio.run(main())
```

## 3.Generic

`typing.Generic` 是 `typing` 模块提供的一个泛型基类，用于创建泛型类。

示例：

```python
from typing import Generic, TypeVar

# 定义泛型变量
T = TypeVar('T')

# 泛型类
class Box(Generic[T]):
    def __init__(self, content: T):
        self.content = content

# 使用泛型类
string_box = Box("Hello, Generic!")
int_box = Box(42)

# 函数使用泛型
def get_content(box: Box[T]) -> T:
    return box.content

string_content = get_content(string_box)
int_content = get_content(int_box)

print(string_content)  # 输出: Hello, Generic!
print(int_content)     # 输出: 42
```

在这个例子中，`Box` 是一个泛型类，使用 `Generic[T]` 定义了类型参数 `T`。通过在实例化时传递具体的类型，我们可以创建包含不同类型内容的盒子。