---
categories:
- - flask
date: 2024-04-19 09:54:05
description: 本篇文章介绍flask框架主要逻辑的简化版实现。web框架的主要逻辑是根据request请求的路由，匹配一个函数并执行后，将其结果添加到response后返回。
id: '161'
tags:
- python
- web框架
title: Flask源码系列1-核心架构
---


本篇文章介绍flask框架主要逻辑的简化版实现。web框架的主要逻辑是根据request请求的路由，匹配一个函数并执行后，将其结果添加到response后返回。

## 1.wsgi

WSGI（Web Server Gateway Interface）是 Python Web 应用程序和 Web 服务器之间的标准接口。`werkzeug.run_simple` 是 Werkzeug 库中用于运行简单 WSGI 应用程序的函数。

以下是 `run_simple` 的基本用法：

```python
from werkzeug.wrappers import Request, Response
from werkzeug.serving import run_simple

# 示例 WSGI 应用程序
def application(environ, start_response):
    request = Request(environ)
    response = Response("Hello, Werkzeug!", content_type='text/plain')
    return response(environ, start_response)

# 运行简单的 HTTP 服务器
if __name__ == '__main__':
    run_simple('localhost', 5000, application)
```

## 2.简化版实现

```python
from werkzeug.wrappers import Request, Response
from werkzeug.serving import run_simple

class Flask():
    def __init__(self) -> None:
        self.route_map = {}  # 路由规则存储

    # 用于添加路由规则
    def route(self, rule: str):
        def wrapper(func):
            self.route_map[rule] = func
            return func
        return wrapper

    # 启动http服务器
    def run(self, host: str, port: int = 5000):
        run_simple(host, port, self, use_reloader=True)

    # wsgi应用
    def __call__(self, environ, start_response):
        request = Request(environ)
        if request.path in self.route_map:
            result = self.route_map[request.path]()
        else:
            result = "Hello, Werkzeug!"

        response = Response(result, content_type='text/plain')
        return response(environ, start_response)

app = Flask()

@app.route("/")
def hello():
    return "Hello, World!"

if __name__ == '__main__':
    app.run('localhost')
```

1.  首先，我们使用route装饰器添加了一个路由规则，将根路径匹配到了hello函数，并保存到了route\_map字典中；
2.  run函数调用werkzeug包的run\_simple函数，启动了一个http服务器；
3.  run\_simple函数的第三个参数是一个可调用的函数，我们在类中实现**call**方法，就可以让实例本身可以被调用，所以第三个参数为self；
4.  在**call**方法中，我们根据request请求拿到请求路径后，在route\_map字典中匹配对应的函数，执行后将结果添加到response并返回。

以上是对flask框架的一个简化版实现，旨在帮助大家理解其核心本质。web框架一般还需要高效的路由分发规则、context上下文的封装、中间件实现等核心部件，这里不再介绍，接下来只介绍flask源码中的python高级特性相关内容。