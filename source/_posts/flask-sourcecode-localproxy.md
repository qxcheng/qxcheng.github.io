---
categories:
- - flask
date: 2024-04-19 10:25:16
description: 本篇是该系列的终篇，为大家介绍flask对werkzeug包中LocalProxy的使用。
id: '180'
tags:
- python
- web框架
title: Flask源码系列7-LocalProxy
---


本篇是该系列的终篇，为大家介绍flask对werkzeug包中LocalProxy的使用。

## 1.contextvars

使用`threading.local()`对象可以基于线程存储全局变量，但不能基于协程。

Python 3.7 引入`contextvars`标准库模块，用于处理上下文变量（Context Variables），它在协程、线程等上下文之间共享，而不会污染全局命名空间。

contextvars的使用：

```python
import asyncio
import contextvars

# 声明Context变量
user_id = contextvars.ContextVar('user_id')
user_ids = contextvars.ContextVar('user_ids')

token = user_id.set(0)  # 设置值
user_id.reset(token)    # 清除设置的值，再调用get()会触发LookupError异常
try:
    print(f'main user_id user_ids: {user_id.get()}')
except LookupError as e:
    print('no value')
    user_id.set(0)

# Context变量的值设为可变对象时，修改后对其他协程是可见的
user_ids.set({})

async def func(user_id_value):
    # 设置值
    user_id.set(user_id_value)
    user_id_dict = user_ids.get()
    user_id_dict[user_id_value] = user_id_value

async def new_coro(user_id_value):
    print(f'coro {user_id_value} before func user_id user_ids: {user_id.get()} {user_ids.get()}')
    await func(user_id_value)
    print(f'coro {user_id_value} after  func user_id user_ids: {user_id.get()} {user_ids.get()}')

async def main():
    tasks = []
    for num in range(1, 4):
        tasks.append(asyncio.create_task(new_coro(num)))

    await asyncio.gather(*tasks)

asyncio.run(main())

print(f'main user_id user_ids: {user_id.get()} {user_ids.get()}')

# 
no value
coro 1 before func user_id user_ids: 0 {}
coro 1 after  func user_id user_ids: 1 {1: 1}
coro 2 before func user_id user_ids: 0 {1: 1}
coro 2 after  func user_id user_ids: 2 {1: 1, 2: 2}
coro 3 before func user_id user_ids: 0 {1: 1, 2: 2}
coro 3 after  func user_id user_ids: 3 {1: 1, 2: 2, 3: 3}
main user_id user_ids: 0 {1: 1, 2: 2, 3: 3}
```

普通函数使用 `contextvars` 模块，需要手动创建上下文：

```python
import contextvars

user_id = contextvars.ContextVar('user_id')
user_id.set(100)

def func(user_id_value):
    user_id.set(user_id_value)
    print("in func user_id:", user_id.get())

def main():
    print('before func user_id:', user_id.get())
    # 创建一个上下文
    context = contextvars.copy_context()
    # 在上下文中运行
    context.run(func, 200)
    print('after func user_id:', user_id.get())

main()

#
before func user_id: 100
in func user_id: 200
after func user_id: 100
```

## 2.LocalProxy

LocalProxy的主要作用是先根据特定的方法，获取代理对象设置的值本身或其某个属性，然后对其做后续的操作。

LocalProxy能代理四种类型的对象：ContextVar、Local、LocalStack、Callable对象。Local和LocalStack是对ContextVar的封装：

*   Local将ContextVar的值设为字典，基于字典的key完成属性的设置和读取
*   LocalStack将ContextVar的值设为列表，提供了push和pop方法

LocalProxy使用示例：

```python
from contextvars import ContextVar
from werkzeug.local import LocalProxy

class G():
    def __getattr__(self, name):
        return 'G'

class A():
    def __init__(self):
        self.name = 'Tom'
        self.g = G()

    def __len__(self):
        return 3

_cv_app = ContextVar("flask.app_ctx")
app_ctx = LocalProxy(_cv_app, unbound_message='_no_app_msg')
g = LocalProxy(_cv_app, 'g', unbound_message='_no_app_msg')

a = A()
_cv_app.set(a)

print(len(app_ctx))  # 3
print(app_ctx.name)  # Tom
print(g.name)        # G
```

## 3.源码分析

Flask使用了ContextVar，源码如下：

```python
# globals.py
_cv_app: ContextVar[AppContext] = ContextVar("flask.app_ctx")  
app_ctx: AppContext = LocalProxy(_cv_app, unbound_message=_no_app_msg)  
current_app: Flask = LocalProxy(_cv_app, "app", unbound_message=_no_app_msg)  
g: _AppCtxGlobals = LocalProxy(_cv_app, "g", unbound_message=_no_app_msg)

_cv_request: ContextVar[RequestContext] = ContextVar("flask.request_ctx")  
request_ctx: RequestContext = LocalProxy(_cv_request, unbound_message=_no_req_msg)  
request: Request = LocalProxy(_cv_request, "request", unbound_message=_no_req_msg)  
session: SessionMixin = LocalProxy(_cv_request, "session", unbound_message=_no_req_msg)

# ctx.py
class AppContext:
    def __init__(self, app: Flask) -> None:  
        self.app = app  
        self.g: _AppCtxGlobals = app.app_ctx_globals_class()  
        self._cv_tokens: list[contextvars.Token] = []

class RequestContext:
    def __init__(self, app, environ, request = None, session = None):
        self.app = app
        self.request: Request = request
        self.session: SessionMixin  None = session
        self._cv_tokens: list[tuple[contextvars.Token, AppContext  None]] = []
```

werkzeug包LocalProxy关键源码提取如下：

```python
class _ProxyLookup:
    def __init__(self, f = None):
        if hasattr(f, "__get__"):
            # A Python function, can be turned into a bound method.
            def bind_f(instance: LocalProxy, obj: t.Any) -> t.Callable:
                return f.__get__(obj, type(obj))  # type: ignore

        self.bind_f = bind_f

    def __get__(self, instance: LocalProxy, owner: type  None = None) -> t.Any:
        try:
            # 调用代理对象的方法
            obj = instance._get_current_object()
        except RuntimeError:
            ...

        if self.bind_f is not None:
            return self.bind_f(instance, obj)

    def __call__(self, instance: LocalProxy, *args, **kwargs):
        return self.__get__(instance, type(instance))(*args, **kwargs)

def _identity(o: T) -> T:
    return o

class LocalProxy():
    def __init__(self, local, name = None, *, unbound_message = None):
        if name is None:
            # 此方法返回自身
            get_name = _identity
        else:
            # 此方法提取name属性
            get_name = attrgetter(name)

        ...
        elif isinstance(local, ContextVar):
            def _get_current_object() -> T:
                try:
                    # 获取ContextVar设置的值
                    obj = local.get()
                except LookupError:
                    raise RuntimeError(unbound_message) from None
                # 返回obj自身或obj.name属性
                return get_name(obj)

        object.__setattr__(self, "_get_current_object", _get_current_object)

    __getattr__ = _ProxyLookup(getattr)
    __setattr__ = _ProxyLookup(setattr)
    __len__ = _ProxyLookup(len)
    ...
```

1.  `__getattr__`是一个描述符，当其被调用时，触发`_ProxyLookup.__call__`方法，该方法调用自身的`__get__`方法
2.  `__get__`方法首先调用代理对象的`_get_current_object`方法，获取代理对象背后的obj对象，然后将普通的`__getattr__`函数包装成obj对象的方法，接着返回这个方法，最后在`_ProxyLookup.__call__`中被调用。

`operator.attrgetter` 是 Python 标准库中 `operator` 模块提供的一个函数，用于创建一个可以提取对象属性的函数。使用示例：

```python
from operator import attrgetter

class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

people = [Person("Alice", 30), Person("Bob", 25), Person("Charlie", 35)]

# 使用 attrgetter 创建一个提取 'age' 属性的函数
get_age = attrgetter('age')

# 对列表中的对象按照 'age' 属性排序
sorted_people = sorted(people, key=get_age)

for person in sorted_people:
    print(person.name, person.age)

#
Bob 25
Alice 30
Charlie 35
```

普通函数的`__get__`方法可以将自己包装成类的方法：

```python
def func(self):
    print(f'2333 {self.name}')

class C:
    name = 123

c = C()

r = func.__get__(c, C)
r()
```

## 4.flask调用链

```python
# app调用run方法，设置启用多线程
app.run(host, port, threaded=True)
    # 调用werkzeug包的run_simple方法
    run_simple(host, port, app, threaded=True)
        # 创建一个server
        srv = make_server(host, port, app, threaded=True)
        # 启动服务，BaseServer方法
        srv.serve_forever() 
            使用select循环处理请求：
                srv._handle_request_noblock()
                    # BaseServer方法，ThreadingMixIn重写
                    srv.process_request(request, client_address) 
                        开启一个线程处理请求：
                        # BaseServer方法
                        srv.finish_request(request, client_address) 
                            srv.RequestHandlerClass(request, client_address, srv) 
                                这里最后执行了BaseRequestHandler的__init__方法，以下将其实例记为wsgi
                                # 跳转到BaseHTTPRequestHandler的方法
                                wsgi.handle() 
                                    wsgi.handle_one_request()
                                    # 跳转到WSGIRequestHandler的方法
                                    wsgi.run_wsgi() 
                                        execute(wsgi.srv.app)
                                        application_iter = app(environ, start_response)

# 上面的最后一步调用了app的__call__方法
app.__call__(environ, start_response)
    app.wsgi_app(environ, start_response)
        # 创建上下文变量
        ctx = self.request_context(environ)
        ctx.push()
            # 设置_cv_request的值，并保存了token
            ctx._cv_tokens.append((_cv_request.set(ctx), app_ctx))
        response = app.full_dispatch_request()
            app.dispatch_request()
                # 这一步跳转到我们编写的视图函数
                # 由于使用了基于ContextVar的LocalProxy，我们可以在视图函数中导入使用全局变量request
                app.ensure_sync(app.view_functions[rule.endpoint])(**view_args)
            app.finalize_request()
        ctx.pop()
            # 恢复_cv_request的值到设置该token之前的状态
            token, app_ctx = ctx._cv_tokens.pop()  
            _cv_request.reset(token)
        return response(environ, start_response)
```

srv继承关系：

```python
srv = ThreadedWSGIServer(host, port, app)
      ThreadedWSGIServer 继承 (ThreadingMixIn, BaseWSGIServer)
      BaseWSGIServer 继承 HTTPServer
      HTTPServer 继承 TCPServer
      TCPServer 继承 BaseServer
```

WSGIRequestHandler继承关系：

```python
RequestHandlerClass = WSGIRequestHandler
WSGIRequestHandler 继承 BaseHTTPRequestHandler 继承 StreamRequestHandler 继承 BaseRequestHandler
```