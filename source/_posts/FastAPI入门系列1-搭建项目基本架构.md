---
categories:
- - fastapi
date: 2024-12-05 20:17:54
description: 'FastAPI 是一个现代、快速（高性能）的 Web 框架，旨在帮助开发者快速构建可靠的 API。基于 Python 的类型注解，FastAPI
  提供了强大的数据验证和文档生成功能，同时支持异步编程，使其在处理高并发请求时表现得尤为出色。得益于其优秀的性能和开发效率，FastAPI 在构建 RESTful
  API 和微服务架构时变得越来越流行。


  在本系列博客中，我们将带领大家逐步了解如何使用 FastAPI 从零开始构建一个功能完备的 API 项目。通过实例演示，我们将深入讲解 FastAPI 的基本概念、常用功能以及最佳实践，帮助你快速上手并掌握这一框架的核心技术。'
id: '351'
tags:
- web框架
title: FastAPI入门系列1-搭建项目基本架构
---


FastAPI 是一个现代、快速（高性能）的 Web 框架，旨在帮助开发者快速构建可靠的 API。基于 Python 的类型注解，FastAPI 提供了强大的数据验证和文档生成功能，同时支持异步编程，使其在处理高并发请求时表现得尤为出色。得益于其优秀的性能和开发效率，FastAPI 在构建 RESTful API 和微服务架构时变得越来越流行。

在本系列博客中，我们将带领大家逐步了解如何使用 FastAPI 从零开始构建一个功能完备的 API 项目。通过实例演示，我们将深入讲解 FastAPI 的基本概念、常用功能以及最佳实践，帮助你快速上手并掌握这一框架的核心技术。

## 1.环境安装

首先，我们通过以下命令安装 FastAPI 及相关依赖包：

```python
pip install fastapi uvicorn SQLAlchemy aiomysql redis gunicorn -i https://pypi.tuna.tsinghua.edu.cn/simple
```

*    `fastapi`：是我们用来构建 API 的核心库。
*    `uvicorn`：FastAPI 的 ASGI 服务器，用于运行应用。
*    `SQLAlchemy`：ORM 库，用于数据库交互。
*    `aiomysql`：一个支持异步操作的 MySQL 客户端。
*    `redis`：用于缓存或队列系统的 Redis 客户端。
*    `gunicorn`：一个高效的 WSGI HTTP 服务器，用于生产环境中运行应用。

## 2.项目文件结构

如下为一个基本的 FastAPI 项目结构，通过组织不同的功能模块和代码，使得项目更易于维护和扩展。

```python
├── api
│   └── users.py      # 用户相关的 API 逻辑
├── forms
│   └── user_form.py  # Pydantic 表单模型，用于数据验证
├── utils
│   ├── depends.py    # 依赖注入工具
│   └── response.py   # HTTP 响应工具类
├── main.py           # FastAPI 应用启动文件
└── requirements.txt  # 项目依赖文件
```

*    `api/` 目录存放路由文件，每个文件负责不同的业务模块。
*    `forms/` 目录存放 Pydantic 数据模型，用于数据验证和序列化。
*    `utils/` 目录存放通用的工具类和函数，如响应工具和依赖注入工具。
*    `main.py` 是 FastAPI 应用的入口，通常在此处启动应用并包含路由。
*    `requirements.txt` 存储所有项目依赖包的列表。

## 3.main.py

在 `main.py` 文件中，我们通过 `FastAPI` 创建应用实例，并将不同的 API 路由（如 `users`）引入到应用中。

```python
from fastapi import FastAPI, Request

from api import users

app = FastAPI()
app.include_router(users.users_router)  # 引用路由
```

*   我们通过 `FastAPI()` 创建了一个 FastAPI 应用实例 `app`。
*   使用 `app.include_router()` 引入了 `users` 路由，这样就可以访问与用户相关的 API 了。

## 4.api/users.py

在 `users.py` 中，我们定义了与用户相关的 API 路由。FastAPI 提供了非常灵活的方式来定义路径参数、查询参数、请求体参数，并支持详细的验证。

*   使用 `APIRouter()` 来定义路由，`prefix` 设置统一的路径前缀，`tags` 用于在文档中进行分组，`responses` 自定义了返回的错误响应。
*   接下来是定义多个具体的 API 路由，包括路径参数、查询参数、请求体参数和依赖注入。

```python
from typing import Optional
from fastapi import APIRouter, Depends, Query, Body, Path

from forms.user_form import UserReq
from utils.depends import default_user
from utils.response import HttpResponse

users_router = APIRouter(
    prefix="/api/v1",  # 路径前缀
    tags=["users"],    # 文档分组
    responses={404: {"description": "Not found"}},  # 自定义某个状态码的响应内容
)

# 路径参数及验证
@users_router.get("/users_v1/{user_id}")
async def get_users_v1(
    user_id: int = Path(
        ..., 
        ge=1,  # 大于等于1（gt lt le ge）
        title="The ID of the user to get"
    )  
):
    return HttpResponse.success(data={"user_id": user_id})

# 查询参数及验证
@users_router.get("/users_v2/")
async def get_users_v2(
    p: str = Query(
        ...,                   # ... 代表必需填
        min_length=1,          # 最短为1
        max_length=20,         # 最长为20
  regex="^@",            # 以@开头
  title="Query string",  # 标题
  description="Query string to search",
  alias="params",        # url查询参数的别名
  deprecated=True,       # 文档中展示为已弃用
    ),  
 q: Optional[str] = Query(None)   # 不是必填，默认为None
):
    result = p + (q if q else '')
    return HttpResponse.success(data={"result": result})

# 请求体参数示例
@users_router.post(
    "/users_v3/",
 tags=["users"],          # 用于文档中的标签分组
 summary="Get an user",   # 简短描述
 description="Get an user detail",  # 详细的描述
    response_model=UserReq,            # 响应数据的模型类型
    response_model_exclude_unset=True,          # 忽略默认参数
    response_model_include={"name", "mobile"},  # 包含（忽略其他的）
 response_description="The user item",       # 对响应内容的描述
 deprecated=True          # 文档中展示为已弃用
)
async def get_users_v3(
    user: UserReq = Body(...)  
):
    return HttpResponse.success(data={"user": user})

# 依赖注入示例
@users_router.get("/users_v4/")
async def get_users_v4(user: UserReq = Depends(default_user)):
    return HttpResponse.success(data={"user": user})
```

## 5.forms/user\_form.py

在 `user_form.py` 文件中，我们通过 Pydantic 创建了一个 `UserReq` 类，用于对请求体数据进行验证和序列化。

*    `UserReq` 类继承自 `BaseModel`，所有字段都会自动验证。
*    `age` 字段使用了 `Field(..., gt=0)`，确保值大于 0。
*    `introduction` 字段是可选的，最大长度为 300 个字符。

```python
from typing import Optional
from pydantic import BaseModel, Field

class UserReq(BaseModel):
    name: str
    mobile: str
    age: int = Field(..., gt=0, description="The age must be greater than zero")
    introduction: Optional[str] = Field(
        None, 
        title="The introduction of the user", 
        max_length=300, 
        example="A very nice man"
    )
```

## 6.utils/depends.py

在 `depends.py` 文件中，我们定义了一个默认的用户对象，使用 FastAPI 的依赖注入系统返回一个 `UserReq` 对象。

`default_user` 函数创建了一个默认的用户对象，返回给调用者。它被用作依赖注入，可以在路由中通过 `Depends()` 获取。

```python
from forms.user_form import UserReq

async def default_user() -> UserReq:
    user = UserReq(
        name='default',
        mobile='123',
        age=18,
        introduction='default'
    )
    return user
```

## 7.utils/response.py

在 `response.py` 文件中，我们定义了一个 `HttpResponse` 工具类，简化了响应的生成过程，并提供了常见的 HTTP 状态码。

*    `HttpResponse` 类提供了多种类型的响应方法：成功 (`success`)、未授权 (`unauth_error`)、参数错误 (`params_error`) 和服务器错误 (`server_error`)。
*    `response()` 方法根据给定的参数构建一个标准的响应格式。

```python
from fastapi import status
from fastapi.encoders import jsonable_encoder
from fastapi.responses import JSONResponse, Response
from typing import Any, Dict, Optional

class HttpResponse:
    """
    响应工具类
    """
    ok = 200
    unautherror = 401
    paramserror = 400
    servererror = 500

    @classmethod
    def response(cls, code: int, msg: str, data: Optional[Any], state: Optional[Any]) -> Response:
        result = {
            "code": code,
            "msg": msg,
            "data": data,
            "state": state
        }
        return JSONResponse(status_code=status.HTTP_200_OK, content=jsonable_encoder(result))

    @classmethod
    def success(
        cls,
        msg: str = '操作成功',
        data: Optional[Dict] = None,
        state: Optional[Any] = None,
    ) -> Response:
        return cls.response(cls.ok, msg, data, state)

    @classmethod
    def unauth_error(
        cls,
        msg: str = '登录信息已过期',
        data: Optional[Dict] = None,
        state: Optional[Any] = None,
    ) -> Response:
        return cls.response(cls.unautherror, msg, data, state)

    @classmethod
    def params_error(
        cls,
        msg: str = '客户端参数错误',
        data: Optional[Dict] = None,
        state: Optional[Any] = None,
    ) -> Response:
        return cls.response(cls.paramserror, msg, data, state)
    

    @classmethod
    def server_error(
        cls,
        msg: str = '服务器内部错误',
        data: Optional[Dict] = None,
        state: Optional[Any] = None,
    ) -> Response:
        return cls.response(cls.servererror, msg, data, state)
```

## 8.测试

使用 `curl` 或浏览器文档来测试我们实现的 API 接口。

*   启动服务后，我们可以访问 `http://127.0.0.1:5000/docs` 来查看自动生成的文档，并且直接在文档中测试 API。
*   使用 `curl` 命令发送请求，测试路径参数验证、查询参数验证和请求体参数。

```python
# 启动服务
uvicorn main:app --reload --port 5000

# 查看文档，可以直接在文档中调试
http://127.0.0.1:5000/docs

# 路径参数验证
curl -X GET 127.0.0.1:5000/api/v1/users_v1/0

{"detail":[{"type":"greater_than_equal","loc":["path","user_id"],"msg":"Input should be greater than or equal to 1","input":"0","ctx":{"ge":1}}]}

# 查询参数及验证
curl -X GET 127.0.0.1:5000/api/v1/users_v2/?params=@hello

{"code":200,"msg":"操作成功","data":{"result":"@hello"},"state":null}

# 请求体参数示例
curl -H "Content-Type: application/json" -X POST -d '{"name":"myname", "mobile":"123", "age": 18}' 127.0.0.1:5000/api/v1/users_v3/

{"code":200,"msg":"操作成功","data":{"user":{"name":"myname","mobile":"123","age":18,"introduction":null}},"state":null}

# 依赖注入示例
curl -X GET 127.0.0.1:5000/api/v1/users_v4/

{"code":200,"msg":"操作成功","data":{"user":{"name":"default","mobile":"123","age":18,"introduction":"default"}},"state":null}
```