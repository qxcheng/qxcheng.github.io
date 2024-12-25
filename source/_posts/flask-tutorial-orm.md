---
categories:
- - flask
date: 2024-04-19 09:43:33
description: 本篇我们介绍ORM框架SQLAlchemy在flask项目中的集成和使用，以此简化数据库的CRUD操作。同时添加连接redis的代码，redis的使用后续另行介绍。
id: '151'
tags:
- python
- web框架
title: Flask教程系列4-数据库ORM
---


本篇我们介绍ORM框架SQLAlchemy在flask项目中的集成和使用，以此简化数据库的CRUD操作。同时添加连接redis的代码，redis的使用后续另行介绍。

## 1.文件结构

如下是本篇完成后的项目文件结构：

```bash
.
├── apps
│   ├── api_v1
│   │   ├── __init__.py
│   │   └── user.py
│   ├── config.py
│   ├── extensions.py
│   ├── forms
│   │   └── user.py
│   ├── __init__.py
│   ├── models
│   │   ├── __init__.py
│   │   └── user.py
│   └── utils
│       └── response.py
├── data
│   ├── mysql
│   │   ├── flask_demo
│   └── redis
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── wsgi.py
```

## 2.requirements.txt

```bash
Flask-SQLAlchemy==3.1.1
Flask-Migrate==4.0.5
pymysql==1.1.0
redis==5.0.1
```

## 3.创建数据库

我们需要先进入mysql容器创建一个数据库，注意检查容器的名称是否和你的一致。

```bash
docker exec -it flasktutorial_mysql_1 /bin/sh

mysql -u root -p
root

CREATE DATABASE flask_demo default charset=utf8mb4;
```

## 4.apps/config.py

在配置文件中填写mysql和redis的配置信息。

```python
class BaseConfig(object):
    SECRET_KEY = "faj$[oSFwdr<2u42@#sddPD"

    # mysql
    SQLALCHEMY_POOL_SIZE = 100
    SQLALCHEMY_POOL_TIMEOUT = 10
    SQLALCHEMY_POOL_RECYCLE = 3600
    SQLALCHEMY_TRACK_MODIFICATIONS = False

class DevConfig(BaseConfig):
    pass

class ProdConfig(BaseConfig):
    # mysql
    MYSQL_HOSTNAME = 'mysql'
    MYSQL_PORT = 3306
    MYSQL_USERNAME = 'root'
    MYSQL_PASSWORD = 'root'
    MYSQL_DATABASE = 'flask_demo'
    SQLALCHEMY_DATABASE_URI = f'mysql+pymysql://{MYSQL_USERNAME}:{MYSQL_PASSWORD}@{MYSQL_HOSTNAME}:{MYSQL_PORT}/{MYSQL_DATABASE}?charset=utf8mb4'

    # redis
    REDIS_HOSTNAME = 'redis'
    REDIS_PORT = 6379
    REDIS_PASSWORD = ''
    REDIS_DB = 15
    REDIS_URL = f'redis://:{REDIS_PASSWORD}@{REDIS_HOSTNAME}:{REDIS_PORT}/{REDIS_DB}'

config = {
    'production': ProdConfig,
    'development': DevConfig
}
```

## 5.apps/extensions.py

在扩展中初始化了SQLAlchemy和Migrate的实例，Migrate后续用于数据库的迁移。这里我们继承了原先的SQLAlchemy类，并添加了一个用于提交和异常回滚的上下文管理器，在视图中我们会使用它。

```python
from contextlib import contextmanager
from flask_cors import CORS
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy as _SQLAlchemy

class SQLAlchemy(_SQLAlchemy):
    @contextmanager
    def auto_commit(self):
        try:
            yield
            self.session.commit()
        except Exception as e:
            self.session.rollback()
            raise e

cors = CORS()
db = SQLAlchemy()
migrate = Migrate()
```

## 6.apps/**init**.py

在创建app时，注册新的拓展，同时创建了一个redis连接池，redis连接池通过注册钩子函数，在每次请求到达视图函数前绑定到对象g上，从而让我们可以在视图函数中使用。

```python
import redis
from flask import Flask, g

from apps.config import config
from apps.api_v1 import api_v1
from apps.extensions import cors, db, migrate

def create_app(config_name=None):
    if config_name is None:
        config_name = 'development'

    app = Flask(__name__)
    app.config.from_object(config[config_name])
    app.secret_key = app.config["SECRET_KEY"]

    # flush 可以在docker环境下立即打印输出
    print(app.config['SQLALCHEMY_DATABASE_URI'], flush=True)

    register_blueprints(app)
    register_extensions(app)
    register_hooks(app)

    with app.app_context():
        db.create_all()  # 根据模型类创建表

    return app

# 注册蓝图
def register_blueprints(app):
    app.register_blueprint(api_v1, url_prefix='/api/v1')

# 注册拓展
def register_extensions(app):
    cors.init_app(app, supports_credentials=True)
    db.init_app(app)
    migrate.init_app(app, db)

    # redis池
    app.redis_pool = redis.ConnectionPool(host=app.config['REDIS_HOSTNAME'],
                                          port=app.config['REDIS_PORT'],
                                          password=app.config['REDIS_PASSWORD'],
                                          db=app.config['REDIS_DB'],
                                          decode_responses=True)

# 注册钩子函数   
def register_hooks(app):

    @app.before_request
    def bind_redis():
        g.redis = redis.Redis(connection_pool=app.redis_pool)
```

## 7.apps/models/**init**.py

```python
from .user import User
```

## 8.apps/models/user.py

这里是user的模型类，其中的mobile字段属于敏感信息，我们通过自定义字段类型进行了加密处理，并为其添加了索引。

```python
from datetime import datetime
from sqlalchemy import cast, func, LargeBinary, type_coerce
from sqlalchemy.dialects.mysql import CHAR
from sqlalchemy.types import TypeDecorator

from apps.extensions import db

my_key = "455dfgfg@#gb)t7"

# 自定义加密类型，用于手机号加密存储
class EncType(TypeDecorator):
    impl = LargeBinary

    def bind_expression(self, bindvalue):
        return func.aes_encrypt(
            type_coerce(bindvalue, CHAR()), func.unhex(func.sha2(my_key, 512)),
        )

    def column_expression(self, col):
        return cast(
            func.aes_decrypt(col, func.unhex(func.sha2(my_key, 512)),),
            CHAR(charset="utf8"),
        )

class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    mobile = db.Column(EncType)
    username = db.Column(db.String(100))
    password = db.Column(db.String(120))
    age = db.Column(db.Integer)
    birth = db.Column(db.DateTime)

    create_time = db.Column(db.DateTime, default=datetime.now)
    update_time = db.Column(db.DateTime, default=datetime.now)
    delete_time = db.Column(db.DateTime, default=datetime.now)

    __table_args__ = (
        db.Index("index_mobile", mobile, mysql_length=16),
    )
```

## 9.apps/api\_v1/user.py

我们对user视图类进行了相应的修改，增加了CRUD操作。

```python
from flask.views import MethodView
from flask import request

from apps.api_v1 import api_v1
from apps.forms.user import SignUpForm
from apps.utils.response import HttpResponse
from apps.extensions import db
from apps.models.user import User

class UserAPI(MethodView):

    def get_user_info(self, user:User):
        return {
            "username": user.username,
            "id": user.id,
            "age": user.age,
            "mobile": user.mobile
        }

    def user_not_found(self, user_id):
        return f"user not found: {user_id}"

    def get(self, user_id:int):
        # get users or get user
        if user_id is None:
            users = User.query.all()
            data = [self.get_user_info(item) for item in users]
            return HttpResponse.success(data={"data": data})
        else:
            user = User.query.get(user_id)
            if not user:
                return HttpResponse.params_error(message=self.user_not_found(user_id))
            else:
                return HttpResponse.success(data=self.get_user_info(user))

    def post(self):
        # add user
        form = SignUpForm.from_json(request.json)
        if form.validate():
            user = User(
                username=form.username.data,
                password=form.password.data,
                mobile=form.mobile.data,
                age=form.info.age.data,
                birth=form.info.birth.data
            )
            with db.auto_commit():
                db.session.add(user)
            return HttpResponse.success(data=self.get_user_info(user))
        else:
            message = f"add user fail:{form.errors}"
            return HttpResponse.params_error(message=message)

    def delete(self, user_id:int):
        # delete user
        user = User.query.get(user_id)
        if not user:
            return HttpResponse.params_error(message=self.user_not_found(user_id))
        else:
            with db.auto_commit():
                db.session.delete(user)
            return HttpResponse.success(message=f'delete user success: {user_id}')

    def put(self, user_id:int):
        # update user
        form = SignUpForm.from_json(request.json)
        if form.validate():
            user = User.query.get(user_id)
            if not user:
                return HttpResponse.params_error(message=self.user_not_found(user_id))

            with db.auto_commit():
                user.username=form.username.data
                user.password=form.password.data
                user.mobile=form.mobile.data
                user.age=form.info.age.data
                user.birth=form.info.birth.data

            return HttpResponse.success(data=self.get_user_info(user))
        else:
            message = f"update user fail:{form.errors}"
            return HttpResponse.params_error(message=message)

user_view = UserAPI.as_view('user_api')
api_v1.add_url_rule('/users/', defaults={'user_id': None}, view_func=user_view, methods=['GET',])
api_v1.add_url_rule('/users/', view_func=user_view, methods=['POST',])
api_v1.add_url_rule('/users/<int:user_id>', view_func=user_view, methods=['GET', 'PUT', 'DELETE'])     
```

## 10.测试

```bash
# 增
curl -H "Content-Type: application/json" -X POST -d '{"username":"12345678", "password":"12345678", "mobile": "13200001111", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:5000/api/v1/users/
{
  "code": 200,
  "data": {
    "age": 18,
    "id": 1,
    "mobile": "13200001111",
    "username": "12345678"
  },
  "message": "SUCCESS",
  "state": 0
}

# 查
curl 127.0.0.1:5000/api/v1/users/
{
  "code": 200,
  "data": {
    "data": [
      {
        "age": 18,
        "id": 1,
        "mobile": "13200001111",
        "username": "12345678"
      }
    ]
  },
  "message": "SUCCESS",
  "state": 0
}

curl 127.0.0.1:5000/api/v1/users/1
{
  "code": 200,
  "data": {
    "age": 18,
    "id": 1,
    "mobile": "13200001111",
    "username": "12345678"
  },
  "message": "SUCCESS",
  "state": 0
}

# 改
curl -H "Content-Type: application/json" -X PUT -d '{"username":"88888888", "password":"12345678", "mobile": "13200001111", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:5000/api/v1/users/1
{
  "code": 200,
  "data": {
    "age": 18,
    "id": 1,
    "mobile": "13200001111",
    "username": "88888888"
  },
  "message": "SUCCESS",
  "state": 0
}

# 删
curl -X DELETE 127.0.0.1:5000/api/v1/users/1
{
  "code": 200,
  "data": {},
  "message": "delete user success: 1",
  "state": 0
}
```