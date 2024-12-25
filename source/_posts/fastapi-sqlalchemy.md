---
categories:
- - fastapi
date: 2024-12-05 21:26:50
description: 本文详细介绍了如何使用 SQLAlchemy 通过异步操作与MySQL数据库进行交互。在实现过程中，我们使用了 FastAPI 提供的生命周期事件，SQLAlchemy
  的异步支持，以及自定义数据库会话管理和事务封装方法。通过这种结构，文章展示了如何在 FastAPI 中组织代码，使得开发、测试和扩展都变得更加高效和清晰。在实际开发中，可以在此基础上继续扩展更多功能。
id: '342'
tags:
- web框架
title: FastAPI入门系列2-连接SQLAlchemy异步数据库
---


本文详细介绍了如何使用 **SQLAlchemy** 通过异步操作与MySQL数据库进行交互。在实现过程中，我们使用了 FastAPI 提供的生命周期事件，SQLAlchemy 的异步支持，以及自定义数据库会话管理和事务封装方法。通过这种结构，文章展示了如何在 FastAPI 中组织代码，使得开发、测试和扩展都变得更加高效和清晰。在实际开发中，可以在此基础上继续扩展更多功能。

## 1.创建数据库

首先在 MySQL 中创建一个名为 `fastapi_demo` 的数据库，并设置默认字符集为 `utf8mb4`。

```python
mysql -u root -p
root

CREATE DATABASE fastapi_demo default charset=utf8mb4;
```

## 2.main.py

这里通过 `FastAPI` 的生命周期事件来初始化和释放资源。我们使用 `@asynccontextmanager` 来定义 `lifespan` 方法，确保在应用启动时初始化数据库资源，在应用关闭时释放资源。

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from api import users
from model import db_init


# 生命周期事件 用于在项目启动前初始化资源 关闭后释放资源
@asynccontextmanager
async def lifespan(app: FastAPI):
    print('服务启动...')
    await db_init()
    yield
    print('服务关闭...')


app = FastAPI(lifespan=lifespan)
app.include_router(users.users_router)  # 引用路由
```

## 3.`model/__init__.py`

新建 `model` 文件夹，在 `model/__init__.py` 文件中，我们配置了数据库引擎和会话管理。这段代码中，我们使用了 `create_async_engine` 来连接数据库，并通过 `sessionmaker` 定义了一个异步的会话生成器，并且通过 `db_init` 方法初始化数据库表结构。此外，`db_session`作为接口的Depends选项用于提供session，`auto_commmit` 作为一个事务封装器，确保数据库操作的原子性，异常发生时可以回滚事务。

```python
import contextlib
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession


Base = declarative_base()

async_uri = 'mysql+aiomysql://root:root@localhost:3306/fastapi_demo?charset=utf8mb4'
async_engine = create_async_engine(
    url=async_uri,
    echo=True,
    pool_size=10,
    max_overflow=30,
    pool_recycle=60 * 30
)
async_session = sessionmaker(async_engine, class_=AsyncSession)


# 数据库初始化表
async def db_init() -> None:
    async with async_engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)


# session生成器 作为fastapi的Depends选项
async def db_session() -> AsyncSession:
    async with async_session() as session:
        return session


# 事务封装
@contextlib.asynccontextmanager
async def auto_commmit(session: AsyncSession):
    try:
        yield session              # 返回会话
        await session.commit()     # 提交事务
    except Exception as e:
        await session.rollback()   # 回滚事务
        raise e
    finally:
        await session.close()      # 关闭会话


from .user import *
```

## 4.model/user.py

在这个文件中，我们定义了 `User` 数据模型。使用 SQLAlchemy 的声明式模型来表示用户数据表。每个 `User` 对象包括用户的基本信息，如 `name`、`mobile`、`age` 和 `introduction`，以及 `create_time` 来记录创建时间。

```python
from datetime import datetime
from sqlalchemy import Column, String, Integer, DateTime

from . import Base


class User(Base):
    __tablename__ = 'user'

    id = Column(Integer, primary_key=True, autoincrement=True, doc="主键")
    name = Column(String(20), index=True)
    mobile = Column(String(11))
    age = Column(Integer)
    introduction = Column(String(100))

    create_time = Column(DateTime, default=datetime.now)
```

## 5.dao/user\_dao.py

新建 `dao` 文件夹，在 `dao/user_dao.py` 中，我们封装了与数据库交互的具体操作方法，包括用户的增、删、查、改等操作。每个方法都通过异步数据库会话 `AsyncSession` 执行 SQLAlchemy 的查询或修改操作。

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import delete, select, update 

from model import auto_commmit
from model.user import User
from forms.user_form import UserReq


class UserDao:
    """
    用户模块数据库操作层
    """

    @classmethod
    async def get_user_dao(cls, db: AsyncSession, user_id: int):
        """
        根据用户名获取用户信息

        :param db: orm对象
        :param user_name: 用户名
        :return: 当前用户名的用户信息对象
        """
        query_user_info = (
            (
                await db.execute(
                    select(User)
                    .where(User.id == user_id)
                )
            )
            .scalars()
            .first()
        )

        return query_user_info
    
    @classmethod
    async def add_user_dao(cls, db: AsyncSession, user: UserReq):
        """
        新增用户数据库操作

        :param db: orm对象
        :param user: 用户对象
        :return:
        """
        async with auto_commmit(db) as session:
            db_user = User(**user.model_dump())
            session.add(db_user)
            await session.flush()

            return db_user.id
    
    @classmethod
    async def update_user_dao(cls, db: AsyncSession, user_id: int, user: UserReq):
        """
        更新用户数据库操作

        :param db: orm对象
        :param user_id: 用户id
        :param user: 用户对象
        :return:
        """
        async with auto_commmit(db) as session:
            await session.execute(
                update(User)
                .where(User.id == user_id)
                .values(**user.model_dump())
            )

    @classmethod
    async def delete_user_dao(cls, db: AsyncSession, user_id: int):
        """
        删除用户数据库操作

        :param db: orm对象
        :param user_id: 用户id
        :return:
        """
        async with auto_commmit(db) as session:
            await session.execute(delete(User).where(User.id == user_id))       
```

## 6.api/users.py

`api/users.py` 定义了用户相关的 API 路由，包括增、删、查、改等操作。通过 `FastAPI` 的 `Depends` 机制来注入数据库会话，从而执行相应的数据库操作。

```python
from sqlalchemy.ext.asyncio import AsyncSession

from dao.user_dao import UserDao
from model import db_session


# 增
@users_router.post("/users/", tags=["users"])
async def post_users(
    user: UserReq = Body(...),
    db: AsyncSession = Depends(db_session)
):
    user_id = await UserDao.add_user_dao(db, user)
    return HttpResponse.success(data={"user_id": user_id})


# 删
@users_router.delete("/users/{user_id}", tags=["users"])
async def delete_users(
    user_id: int = Path(...) ,
    db: AsyncSession = Depends(db_session)  
):
    await UserDao.delete_user_dao(db, user_id)
    return HttpResponse.success()


# 查
@users_router.get("/users/{user_id}", tags=["users"])
async def get_users(
    user_id: int = Path(...),
    db: AsyncSession = Depends(db_session)  
):
    user = await UserDao.get_user_dao(db, user_id)
    return HttpResponse.success(data={"user": user})


# 改
@users_router.put("/users/{user_id}", tags=["users"])
async def put_users(
    user_id: int = Path(...),
    user: UserReq = Body(...),
    db: AsyncSession = Depends(db_session)  
):
    await UserDao.update_user_dao(db, user_id, user)
    return HttpResponse.success()
```

## 7.测试

在最后的测试部分，我们启动 FastAPI 服务并通过 `curl` 命令进行接口测试。这里展示了如何进行用户的增、删、查、改操作，验证了接口是否正常工作。

```python
# 启动服务
uvicorn main:app --reload --port 5000

# 增
curl -H "Content-Type: application/json" -X POST -d '{"name":"myname", "mobile":"123", "age": 18, "introduction": "student"}' 127.0.0.1:5000/api/v1/users/

{"code":200,"msg":"操作成功","data":{"user_id":5},"state":null}

# 改
curl -H "Content-Type: application/json" -X PUT -d '{"name":"yourname", "mobile":"123", "age": 18, "introduction": "student"}' 127.0.0.1:5000/api/v1/users/5

{"code":200,"msg":"操作成功","data":null,"state":null}

# 查
curl -X GET 127.0.0.1:5000/api/v1/users/5

{"code":200,"msg":"操作成功","data":{"user":{"name":"yourname","mobile":"123","introduction":"student","age":18,"id":5,"create_time":"2024-11-06T22:35:12"}}
 
# 删
curl -X DELETE 127.0.0.1:5000/api/v1/users/5

{"code":200,"msg":"操作成功","data":null,"state":null}
```