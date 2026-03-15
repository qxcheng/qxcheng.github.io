---
categories:
- - fastapi
date: 2024-12-06 20:51:57
description: 这篇博客介绍了如何在 FastAPI 项目中实现全局的中间件和异常处理机制，以及如何进行部署优化。文章详细展示了如何在应用中集成跨域支持、请求处理时间统计等中间件，并通过自定义异常类和异常处理函数来增强错误管理。此外，介绍了如何在部署阶段配置
  Gunicorn 服务器以确保服务的高效运行，最终实现一个健壮且易于扩展的 FastAPI 项目架构。
id: '349'
tags:
- web框架
title: FastAPI入门系列4-使用中间件、捕获异常和项目部署
---


这篇博客介绍了如何在 FastAPI 项目中实现全局的中间件和异常处理机制，以及如何进行部署优化。文章详细展示了如何在应用中集成跨域支持、请求处理时间统计等中间件，并通过自定义异常类和异常处理函数来增强错误管理。此外，介绍了如何在部署阶段配置 Gunicorn 服务器以确保服务的高效运行，最终实现一个健壮且易于扩展的 FastAPI 项目架构。

## 1.项目文件结构

这部分展示了 FastAPI 项目的文件夹结构，涵盖了项目的各个模块，包括 API 路由（`api`）、数据访问层（`dao`）、异常处理（`exceptions`）、表单验证（`forms`）、中间件（`middlewares`）、数据库模型（`model`）、工具类（`utils`）等。清晰的目录结构帮助开发者快速理解项目的组织方式，确保模块化和可维护性。

```python
├── api
│   └── users.py
├── dao
│   └── user_dao.py
├── exceptions
│   ├── exception.py
│   └── handle.py
├── forms
│   └── user_form.py
├── middlewares
│   ├── handle.py
│   └── middleware.py
├── model
│   ├── __init__.py
│   └── user.py
├── utils
│   ├── async_redis.py
│   ├── depends.py
│   └── response.py
├── main.py
└── requirements.txt
```

## 2.main.py

`main.py` 文件集成了自定义的异常处理和中间件。`handle_exception` 和 `handle_middleware` 分别负责处理全局的异常和中间件，提升了系统的可扩展性和错误响应的统一性。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from api import users
from model import db_init
from utils.async_redis import RedisUtil
from exceptions.handle import handle_exception
from middlewares.handle import handle_middleware


# 生命周期事件 用于在项目启动前初始化资源 关闭后释放资源
@asynccontextmanager
async def lifespan(app: FastAPI):
    print('服务启动...')
    await db_init()
    app.state.redis = await RedisUtil.create_redis_pool()
    yield
    print('服务关闭...')


app = FastAPI(lifespan=lifespan)
app.include_router(users.users_router)  # 引用路由


handle_exception(app)
handle_middleware(app)
```

## 3.中间件

### middlewares/middleware.py

`middleware.py` 中定义了两个常见的中间件：

*    **跨域中间件**（CORS）允许指定的前端应用跨域访问 FastAPI 后端 API。通过设置 `allow_origins` 来限制允许的前端来源。
*    **处理时间中间件**：用于计算每个请求的处理时间，并将其以 `X-Process-Time` 头部返回，帮助开发者了解每个请求的处理效率。

```python
import time
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware


def add_cors_middleware(app: FastAPI):
    """
    添加跨域中间件

    :param app: FastAPI对象
    :return:
    """
    # 前端页面url
    origins = [
        'http://localhost:5000',
        'http://127.0.0.1:5000',
    ]

    # 后台api允许跨域
    app.add_middleware(
        CORSMiddleware,
        allow_origins=origins,
        allow_credentials=True,
        allow_methods=['*'],
        allow_headers=['*'],
    )


def add_process_time_middleware(app: FastAPI):

    # 中间件
    @app.middleware("http")
    async def add_process_time_header(request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        print('处理时间：', process_time)
        response.headers["X-Process-Time"] = str(process_time)
        return response
```

### middlewares/handle.py

`handle.py` 是一个中间件处理的集中管理模块，调用了 `add_cors_middleware` 和 `add_process_time_middleware` 两个函数来注册中间件。这个文件的目的是使中间件的管理更加模块化和清晰。

```python
from fastapi import FastAPI
from middlewares.middleware import add_cors_middleware, add_process_time_middleware


def handle_middleware(app: FastAPI):
    """
    全局中间件处理
    """
    # 加载跨域中间件
    add_cors_middleware(app)
    # 加载处理时间中间件
    add_process_time_middleware(app)
```

## 4.异常处理

### exceptions/exception.py

`exception.py` 文件定义了一个自定义的 `LoginException` 类，用于处理与登录相关的特定异常。这个异常类可以带有错误数据和详细的错误消息，帮助开发者更好地捕获和诊断特定的错误场景。

```python
from typing import Optional


class LoginException(Exception):
    """
    自定义登录异常LoginException
    """

    def __init__(self, data: Optional[str] = None, message: Optional[str] = None):
        self.data = data
        self.message = message
```

### exceptions/handle.py

`handle.py` 负责注册全局异常处理器。它处理了：

*    自定义的 `LoginException` 异常，返回特定的认证错误消息。
*    `RequestValidationError` 异常，捕获数据验证失败时的错误，并返回规范化的错误信息。
*    `HTTPException` 异常，处理 HTTP 错误并返回状态码和错误详情。
*    `Exception` 异常，捕获其他未知错误并返回通用的错误响应。

```python
from fastapi import FastAPI, Request
from fastapi.exceptions import HTTPException, RequestValidationError

from exceptions.exception import LoginException
from utils.response import HttpResponse


def handle_exception(app: FastAPI):
    """
    全局异常处理
    """

    # 自定义登录异常
    @app.exception_handler(LoginException)
    async def auth_exception_handler(request: Request, exc: LoginException):
        return HttpResponse.unauth_error(data={'login_error': exc.data}, msg=exc.message)
    
    # 自定义数据验证异常处理
    @app.exception_handler(RequestValidationError)
    async def validation_exception_handler(request: Request, exc: RequestValidationError):
        return HttpResponse.params_error(
            data={
                "error": "Validation failed",
                "details": exc.errors(),
                "message": "Invalid input data. Please correct the fields and try again.",
            },
        )
    
    # 处理其他http请求异常
    @app.exception_handler(HTTPException)
    async def http_exception_handler(request: Request, exc: HTTPException):
        return HttpResponse.server_error(data={'code': exc.status_code, 'msg': exc.detail})

    # 处理其他异常
    @app.exception_handler(Exception)
    async def exception_handler(request: Request, exc: Exception):
        return HttpResponse.server_error(msg=str(exc))
```

## 5.项目部署

在部署部分，文章介绍了如何使用 Gunicorn 启动 FastAPI 应用，并通过配置 `-w` 参数设置适当数量的 worker 进程，以便在生产环境中提高应用的处理能力。`-w` 参数的推荐值是根据系统的 CPU 核心数来确定的：`CPU * 2 + 1`。此外，提供了测试命令和预期的错误响应，演示了如何通过中间件和异常处理机制返回详细的错误信息。

```python
# 启动服务
gunicorn main:app -b 127.0.0.1:5000 -n my_app -w 4 -k uvicorn.workers.UvicornWorker

# 测试
curl -X GET 127.0.0.1:5000/api/v1/users_v1/0

客户端触发参数验证异常，包装后返回：
{"code":400,"msg":"客户端参数错误","data":{"error":"Validation failed","details":[{"type":"greater_than_equal","loc":["path","user_id"],"msg":"Input should be greater than or equal to 1","input":"0","ctx":{"ge":1}}],"message":"Invalid input data. Please correct the fields and try again."},"state":null}

服务端中间件打印：
处理时间： 0.0005788803100585938
```