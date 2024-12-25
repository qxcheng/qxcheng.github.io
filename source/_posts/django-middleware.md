---
categories:
- - django
date: 2024-11-12 15:18:34
description: Django 中间件是一种处理请求和响应的轻量级插件系统，能够在请求到达视图之前或响应返回客户端之前执行特定的处理逻辑。通过中间件，开发者可以实现认证、日志记录、请求处理、异常捕获等功能。本文将介绍如何创建和使用
  Django 中间件。
id: '336'
tags:
- web框架
title: Django入门系列7-使用中间件
---


Django 中间件是一种处理请求和响应的轻量级插件系统，能够在请求到达视图之前或响应返回客户端之前执行特定的处理逻辑。通过中间件，开发者可以实现认证、日志记录、请求处理、异常捕获等功能。本文将介绍如何创建和使用 Django 中间件。

## 1.中间件的基本概念

中间件是一个处理请求和响应的钩子。Django 提供了一系列内置中间件，也允许开发者自定义中间件，以满足特定的需求。中间件的工作流程如下：

1.  请求进入 Django 应用时，中间件会按顺序执行处理请求的方法。
2.  视图处理请求并返回响应。
3.  响应再次经过中间件，按相反的顺序执行处理响应的方法。

## 2.myapp/middleware.py

要创建一个自定义中间件，可以在应用的目录下创建一个新的 Python 文件，假设命名为 `middleware.py`。以下是一个简单的示例，中间件用于记录请求的路径和方法以及花费的时间。

```python
import time


class LoggingMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        # 在视图处理请求之前执行的逻辑
        print(f'请求路径: {request.path}, 方法: {request.method}', flush=True)
        start = time.time()

        response = self.get_response(request)

        # 在视图处理请求之后执行的逻辑
        end = time.time()
        print('响应状态码:', response.status_code, flush=True)
        print("请求花费时间: {}秒".format(end - start), flush=True)
        
        return response
```

## 3.myproject/settings.py

创建完自定义中间件后，需要在 `settings.py` 文件中进行注册。在 `MIDDLEWARE` 列表中添加中间件的路径：

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'myapp.middleware.LoggingMiddleware',  # 注册自定义中间件
]
```

## 4.测试

```python
# docker-compose重新构建web
docker-compose up -d --build web

# 查看服务端docker日志
docker-compose logs web -f

# 获取一篇文章，此时查询数据库
curl -X GET 127.0.0.1:5000/api/v1/articles/1
服务端打印：
django-tutorial-web-1   请求路径: /api/v1/articles/1, 方法: GET
django-tutorial-web-1   使用缓存查询...
django-tutorial-web-1   响应状态码: 200
django-tutorial-web-1   请求花费时间: 0.008148193359375秒
```

## 5.内置中间件

*    `SecurityMiddleware`：为request/response提供了几种安全改进;
*    `CommonMiddleware`：基于APPEND\_SLASH和PREPEND\_WWW的设置来重写URL，如果APPEND\_SLASH设为True，并且初始URL 没有以斜线结尾以及在URLconf 中没找到对应定义，这时形成一个斜线结尾的新URL；
*    `SessionMiddleware`：开启session会话支持；
*    `CsrfViewMiddleware`：添加跨站点请求伪造的保护，通过向POST表单添加一个隐藏的表单字段，并检查请求中是否有正确的值；
*    `AuthenticationMiddleware`：在视图函数执行前向每个接收到的user对象添加HttpRequest属性，表示当前登录的用户，无它用不了`request.user`。
*    `MessageMiddleware`：开启基于Cookie和会话的消息支持
*    `XFrameOptionsMiddleware`：对点击劫持的保护

除此以外, Django还提供了压缩网站内容的`GZipMiddleware`，根据用户请求语言返回不同内容的`LocaleMiddleware`和给GET请求附加条件的`ConditionalGetMiddleware`。这些中间件都是可选的。 如果你自定义的中间件有依赖于`request.user`，那么你自定义的中间件一定要放在`AuthenticationMiddleware`的后面。