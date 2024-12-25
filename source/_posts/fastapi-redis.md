---
categories:
- - fastapi
date: 2024-12-06 16:32:17
description: 这篇博客介绍了如何在 FastAPI 项目中集成 Redis 缓存，以提高数据查询的效率和性能。通过利用 async_redis 工具类和生命周期事件管理，文章展示了如何在应用启动时初始化
  Redis 连接池，并在服务关闭时释放资源。利用 Redis 缓存，API 路由可以在查询数据库时先检查缓存，若缓存存在则直接返回缓存数据，否则从数据库中获取并存入缓存，从而实现缓存优化和性能提升。
id: '345'
tags:
- web框架
title: FastAPI入门系列3-连接异步redis
---


这篇博客介绍了如何在 FastAPI 项目中集成 Redis 缓存，以提高数据查询的效率和性能。通过利用 `async_redis` 工具类和生命周期事件管理，文章展示了如何在应用启动时初始化 Redis 连接池，并在服务关闭时释放资源。利用 Redis 缓存，API 路由可以在查询数据库时先检查缓存，若缓存存在则直接返回缓存数据，否则从数据库中获取并存入缓存，从而实现缓存优化和性能提升。

## 1.main.py

`main.py` 文件展示了 FastAPI 应用的初始化和生命周期管理。在这个文件中，通过使用 `@asynccontextmanager` 装饰器定义了一个生命周期事件（`lifespan`），该事件会在应用启动时初始化数据库和 Redis 连接池，并在服务关闭时释放资源。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from api import users
from model import db_init
from utils.async_redis import RedisUtil


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
```

## 2.utils/async\_redis.py

在 `async_redis.py` 文件中，封装了与 Redis 相关的操作，提供了 Redis 连接池的创建、关闭和缓存操作的实现。具体包括：

*    `create_redis_pool`：应用启动时初始化 Redis 连接池。
*    `close_redis_pool`：应用关闭时关闭 Redis 连接。
*    `get_cache`、`set_cache` 和 `delete_cache`：用于获取、设置和删除 Redis 缓存。

```python
from typing import Any
from fastapi import FastAPI, Request
from redis import asyncio as aioredis
from redis.exceptions import AuthenticationError, TimeoutError, RedisError


class RedisUtil:
    """
    Redis相关方法
    """

    @classmethod
    async def create_redis_pool(cls) -> aioredis.Redis:
        """
        应用启动时初始化redis连接

        :return: Redis连接对象
        """

        redis = await aioredis.from_url(
            url=f'redis://localhost',
            port=6379,
            #username='root',
            #password='password',
            db=1,
            encoding='utf-8',
            decode_responses=True,
        )
        try:
            connection = await redis.ping()
            if connection:
                print('redis连接成功')
            else:
                print('redis连接失败')
        except AuthenticationError as e:
            print(f'redis用户名或密码错误，详细错误信息：{e}')
        except TimeoutError as e:
            print(f'redis连接超时，详细错误信息：{e}')
        except RedisError as e:
            print(f'redis连接错误，详细错误信息：{e}')
        return redis

    @classmethod
    async def close_redis_pool(cls, app: FastAPI):
        """
        应用关闭时关闭redis连接

        :param app: fastapi对象
        :return:
        """
        await app.state.redis.close()
    
    @classmethod
    async def get_cache(cls, request: Request, cache_key: str) -> Any:
        """
        获取缓存内容信息

        :param request: Request对象
        :param cache_key: 缓存键名
        :return: 缓存内容信息
        """
        cache_value = await request.app.state.redis.get(cache_key)
        return cache_value
    
    @classmethod
    async def set_cache(cls, request: Request, cache_key: str, cache_value: str):
        """
        设置缓存内容信息

        :param request: Request对象
        :param cache_key: 缓存键名
        :return:
        """
        await request.app.state.redis.set(cache_key, cache_value)

    @classmethod
    async def delete_cache(cls, request: Request, cache_key: str):
        """
        删除缓存内容信息

        :param request: Request对象
        :param cache_key: 缓存键名
        :return: 
        """
        await request.app.state.redis.delete(cache_key)
```

## 3.api/users.py

`users.py` 处理与用户数据相关的 API 请求。在 `get_users` 路由中，首先会检查 Redis 缓存中是否有对应用户的数据。如果缓存命中，则直接返回缓存数据；若未命中，则从数据库获取用户数据并存入缓存。这种缓存机制能显著减少数据库访问，提高接口响应速度。

```python
from fastapi import Request
from utils.async_redis import RedisUtil

# 查
@users_router.get("/users/{user_id}", tags=["users"])
async def get_users(
    request: Request,
    user_id: int = Path(...),
    db: AsyncSession = Depends(db_session),
):
    cache_key = f"USER::{user_id}"
    username = await RedisUtil.get_cache(request, cache_key)
    if not username:
        print("使用数据库...")
        user = await UserDao.get_user_dao(db, user_id)
        username = user.name
        await RedisUtil.set_cache(request, cache_key, username)
    else:
        print("使用缓存...")
    return HttpResponse.success(data={"username": username})
```

## 4.测试

使用 `curl` 工具发送 HTTP 请求，验证缓存机制的有效性。首次请求时，服务器会从数据库获取用户数据，并存入 Redis 缓存；而第二次请求时，服务器直接从 Redis 获取数据，避免了不必要的数据库查询，验证了缓存的作用。

```python
# 启动服务
uvicorn main:app --reload --port 5000

# 查
curl -X GET 127.0.0.1:5000/api/v1/users/1

服务端打印：
使用数据库...

# 查
curl -X GET 127.0.0.1:5000/api/v1/users/1

服务端打印：
使用缓存...
```