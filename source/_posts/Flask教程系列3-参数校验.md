---
categories:
- - flask
date: 2024-04-19 09:41:48
description: 本篇内容介绍如何对接口的入参做参数校验。
id: '149'
tags:
- python
- web框架
title: Flask教程系列3-参数校验
---


本篇内容介绍如何对接口的入参做参数校验。

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
│   └── utils
│       └── response.py
├── data
│   ├── mysql
│   └── redis
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── wsgi.py
```

## 2.requirements.txt

```bash
Flask==3.0.0
Flask-Cors==4.0.0
WTForms==3.1.0
WTForms-JSON==0.3.5
```

## 3.apps/forms/user.py

下面是我们的参数校验代码，主要用于添加用户时的参数校验。这里实现了参数的嵌套验证和自定义的字段验证。

```python
from wtforms import Form, StringField, FormField, IntegerField, DateTimeField
from wtforms.validators import DataRequired, Length, ValidationError, NumberRange
import wtforms_json

wtforms_json.init()

class InfoForm(Form):
    age = IntegerField('age', validators=[DataRequired(), NumberRange(18, 30)])
    birth = DateTimeField('birth', validators=[DataRequired()])

class SignUpForm(Form):
    username = StringField('username', validators=[DataRequired()])
    password = StringField('password', validators=[DataRequired(), Length(8, 128)])
    info = FormField(InfoForm)

    # 自动调用验证 username 字段
    def validate_username(self, field):  
        print(field.data, flush=True)
        if len(field.data) <= 6:
            raise ValidationError('Username should at least have more than 6 letters!')
```

## 4.apps/api\_v1/user.py

修改视图函数以验证入参的合法性：

```python
from flask.views import MethodView
from flask import request

from apps.api_v1 import api_v1
from apps.forms.user import SignUpForm
from apps.utils.response import HttpResponse

class UserAPI(MethodView):

    def get(self, user_id):
        # get users or get user
        if user_id is None:
            return HttpResponse.success(message="get user list")
        else:
            message = "get user: {}".format(user_id)
            return HttpResponse.success(message=message)

    def post(self):
        # add user
        form = SignUpForm.from_json(request.json)
        if form.validate():
            message = f"add user:{form.username.data}/{form.info.age.data}/{form.info.birth.data}"
            return HttpResponse.success(message=message)
        else:
            message = f"add user fail:{form.errors}"
            return HttpResponse.params_error(message=message)

    def delete(self, user_id):
        # delete user
        message = "delete user: {}".format(user_id)
        return HttpResponse.success(message=message)

    def put(self, user_id):
        # update user
        message = "update user: {}".format(user_id)
        return HttpResponse.success(message=message)

user_view = UserAPI.as_view('user_api')
api_v1.add_url_rule('/users/', defaults={'user_id': None}, view_func=user_view, methods=['GET',])
api_v1.add_url_rule('/users/', view_func=user_view, methods=['POST',])
api_v1.add_url_rule('/users/<int:user_id>', view_func=user_view, methods=['GET', 'PUT', 'DELETE'])     
```

## 5.apps/utiils/response.py

这里是我们封装的统一回复代码。

```python
from flask import jsonify

class HttpResponse(object):
    ok = 200
    redirect = 300
    unautherror = 401
    paramserror = 400
    servererror = 500

    @staticmethod
    def response(code, message='', data={}, state=0):
        return jsonify({"code": code, "message": message, "data": data, "state": state})

    @staticmethod
    def success(message="", data={}, state=0):
        message = message or "SUCCESS"
        return HttpResponse.response(code=HttpResponse.ok, message=message, data=data, state=state)

    @staticmethod
    def redirect(message="", data={}, state=0):
        message = message or "REDIRECT"
        return HttpResponse.response(code=HttpResponse.redirect, message=message, data=data, state=state)

    @staticmethod
    def unauth_error(message="", data={}, state=0):
        message = message or 'UNAUTHORIZED_ERROR'
        return HttpResponse.response(code=HttpResponse.unautherror, message=message, data=data, state=state)

    @staticmethod
    def params_error(message="", data={}, state=0):
        message = message or 'CLIENT_ERROR'
        return HttpResponse.response(code=HttpResponse.paramserror, message=message, data=data, state=state)

    @staticmethod
    def server_error(message="", data={}, state=0):
        message = message or 'SERVER_ERROR'
        return HttpResponse.response(code=HttpResponse.servererror, message=message, data=data, state=state)
```

## 6.apps/extensions.py

这里添加了一个防止跨域报错的扩展。

```python
from flask_cors import CORS

cors = CORS()
```

## 7.apps/**init**.py

下面注册新的扩展：

```python
from flask import Flask

from apps.config import config
from apps.api_v1 import api_v1
from apps.extensions import cors

def create_app(config_name=None):
    if config_name is None:
        config_name = 'development'

    app = Flask(__name__)
    app.config.from_object(config[config_name])
    app.secret_key = app.config["SECRET_KEY"]

    register_blueprints(app)
    register_extensions(app)

    return app

# 注册蓝图
def register_blueprints(app):
    app.register_blueprint(api_v1, url_prefix='/api/v1')

# 注册拓展
def register_extensions(app):
    cors.init_app(app, supports_credentials=True)
```

## 8.测试

```bash
curl -H "Content-Type: application/json" -X POST -d '{"username":"12345678", "password":"12345678", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:5000/api/v1/users/
{
  "code": 400,
  "data": {},
  "message": "add user fail:{'mobile': ['This field is required.']}",
  "state": 0
}

curl -H "Content-Type: application/json" -X POST -d '{"username":"12345678", "password":"12345678", "mobile": "12345678", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:5000/api/v1/users/
{
  "code": 200,
  "data": {},
  "message": "add user:12345678/18/2023-08-23 12:30:59",
  "state": 0
}
```