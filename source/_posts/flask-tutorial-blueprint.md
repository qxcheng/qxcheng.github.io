---
categories:
- - flask
date: 2024-04-19 09:38:52
description: 本篇我们介绍flask中用蓝图组织路由的用法，这样可以使整个项目看起来整洁有序。
id: '146'
tags:
- python
- web框架
title: Flask教程系列2-蓝图组织
---


本篇我们介绍flask中用蓝图组织路由的用法，这样可以使整个项目看起来整洁有序。

## 1.文件结构

如下是本篇完成后的项目文件结构：

```bash
.
├── apps
│   ├── api_v1
│   │   ├── __init__.py
│   │   └── user.py
│   ├── config.py
│   └── __init__.py
├── data
│   ├── mysql
│   └── redis
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── wsgi.py
```

## 2.wsgi.py

现在wsgi中只做创建app的操作，通过传参使用相应的配置参数：

```python
from apps import create_app

app = create_app("production")

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000, debug=True, use_reloader=True)
```

## 3.apps/config.py

如下是我们的配置文件，后续相关的配置都会写在这里。

```python
class BaseConfig(object):
    SECRET_KEY = "faj$[oSFwdr<2u42@#sddPD"

class DevConfig(BaseConfig):
    pass

class ProdConfig(BaseConfig):
    pass

config = {
    'production': ProdConfig,
    'development': DevConfig
}
```

## 4.apps/**init**.py

下面是我们创建flask app的初始化代码，目前主要做了加载配置文件和注册蓝图的操作。

```python
from flask import Flask

from apps.config import config
from apps.api_v1 import api_v1

def create_app(config_name=None):
    if config_name is None:
        config_name = 'development'

    app = Flask(__name__)
    app.config.from_object(config[config_name])
    app.secret_key = app.config["SECRET_KEY"]

    register_blueprints(app)

    return app

# 注册蓝图
def register_blueprints(app):
    app.register_blueprint(api_v1, url_prefix='/api/v1')
```

## 5.apps/api\_v1/**init**.py

下面是创建一个蓝图，并导入user中的视图。

```python
from flask import Blueprint

api_v1 = Blueprint('api_v1', __name__)

from apps.api_v1.user import *
```

## 6.apps/api\_v1/user.py

这里是我们的user视图类，代码最后几行为视图设置路由，并关联到蓝图上。

```python
from flask.views import MethodView
from flask import request

from apps.api_v1 import api_v1

class UserAPI(MethodView):

    def get(self, user_id):
        # get users or get user
        if user_id is None:
            return "get user list"
        else:
            return f"get user: {user_id}"

    def post(self):
        # add user
        username = request.form['username']
        password = request.form['password']

        return f"add user: {username}"

    def delete(self, user_id):
        # delete user
        return f"delete user: {user_id}"

    def put(self, user_id):
        # update user
        return f"update user: {user_id}"

user_view = UserAPI.as_view('user_api')
api_v1.add_url_rule('/users/', defaults={'user_id': None}, view_func=user_view, methods=['GET',])
api_v1.add_url_rule('/users/', view_func=user_view, methods=['POST',])
api_v1.add_url_rule('/users/<int:user_id>', view_func=user_view, methods=['GET', 'PUT', 'DELETE'])    
```

## 7.测试

重启构建项目后，测试应看到如下预期效果：

```bash
curl 127.0.0.1:5000/api/v1/users/
get user list

curl 127.0.0.1:5000/api/v1/users/1
get user: 1

curl -X POST -d 'username=张三&password=123' 127.0.0.1:5000/api/v1/users/
add user: 张三

curl -X DELETE 127.0.0.1:5000/api/v1/users/1
delete user: 1

curl -X PUT 127.0.0.1:5000/api/v1/users/1
update user: 1
```