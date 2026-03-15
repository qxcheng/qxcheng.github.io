---
categories:
- - flask
date: 2024-04-19 09:47:52
description: 本篇文章介绍JWT认证在flask中的使用。
id: '157'
tags:
- python
- web框架
title: Flask教程系列7-JWT认证
---


本篇文章介绍JWT认证在flask中的使用。

## 1.requirements.txt

```yml
Flask-JWT-Extended==4.5.3
```

## 2.apps/extensions.py

新增jwt扩展：

```python
from flask_jwt_extended import JWTManager

jwt = JWTManager()
```

## 3.apps/**init**.py

在创建app时注册新的拓展：

```python
from apps.extensions import jwt

# 注册拓展
def register_extensions(app):
    ...
    jwt.init_app(app)
```

## 4.apps/config.py

配置JWT的key、access\_token过期时间、refresh\_token过期时间，refresh\_token用于获取新的access\_token。

```python
from datetime import timedelta

class BaseConfig(object):
    ...

    JWT_SECRET_KEY = "123twe@FwfrdafwsfsLLD@#"
    JWT_ACCESS_TOKEN_EXPIRES = timedelta(minutes=2)
    JWT_REFRESH_TOKEN_EXPIRES = timedelta(minutes=5)
```

## 5.apps/api\_v1/user.py

在user视图中使用jwt\_required装饰器声明该视图函数需要验证头部中的access\_token，使用get\_jwt\_identity函数获取token中的user\_id信息，create\_access\_token和create\_refresh\_token用于创建token。另外增加了refresh视图函数用于获取新的access\_token，其需要验证头部中的refresh\_token。

```python
from flask_jwt_extended import create_access_token, create_refresh_token, jwt_required, get_jwt_identity

class UserAPI(MethodView):
    ...
    @jwt_required()
    def get(self):
        # get users or get user
        user_id = get_jwt_identity()

        if user_id is None:
            users = User.query.all()
            data = [self.get_user_info(item) for item in users]
            return HttpResponse.success(data={"data": data})
        else:
            data = self.get_user_info(user_id)
            if not data:
                return HttpResponse.params_error(message=self.user_not_found(user_id))
            else:
                return HttpResponse.success(data=data)

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
            self.add_cache(self.extract_user_info(user))

            access_token = create_access_token(identity=user.id)
            refresh_token = create_refresh_token(identity=user.id)
            data = self.get_user_info(user)
            data['access_token'] = access_token
            data['refresh_token'] = refresh_token

            return HttpResponse.success(data=data)
        else:
            message = f"add user fail:{form.errors}"
            return HttpResponse.params_error(message=message)

# 使用refresh_token获取access_token
@api_v1.route("/refresh", methods=["POST"])
@jwt_required(refresh=True)
def refresh():
    identity = get_jwt_identity()
    access_token = create_access_token(identity=identity)
    return HttpResponse.success(data={'access_token': access_token})

...
# 修改两条路由
api_v1.add_url_rule('/users/', view_func=user_view, methods=['GET',])
api_v1.add_url_rule('/users/<int:user_id>', view_func=user_view, methods=['PUT', 'DELETE'])
```

## 6.测试

```python
# 增
curl -H "Content-Type: application/json" -X POST -d '{"username":"12345678", "password":"12345678", "mobile": "13200001111", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:5000/api/v1/users/
{
  "code": 200,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYwOTExMywianRpIjoiYzE2YjI1ZjItMjIyYi00MjliLWFkOGMtODBjMWQxODRjZjRiIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6MiwibmJmIjoxNjk3NjA5MTEzLCJleHAiOjE2OTc2MDkyMzN9.4d2qHwN6_I9H-FavmGDz7Eu9SWRXAnxBH0W7PAG9OTo",
    "age": 18,
    "id": 2,
    "mobile": "13200001111",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYwOTExMywianRpIjoiNTBmNGM2NTUtYWNkNC00ZjAxLTg3NWItNTYwYjIxY2VlMTY5IiwidHlwZSI6InJlZnJlc2giLCJzdWIiOjIsIm5iZiI6MTY5NzYwOTExMywiZXhwIjoxNjk3NjA5NDEzfQ.PnqLxzlDEgzwlrTNhsBE7xzTPGUR8jOaceRHiMEA-Y0",
    "username": "12345678"
  },
  "message": "SUCCESS",
  "state": 0
}

# 查
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYwOTExMywianRpIjoiYzE2YjI1ZjItMjIyYi00MjliLWFkOGMtODBjMWQxODRjZjRiIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6MiwibmJmIjoxNjk3NjA5MTEzLCJleHAiOjE2OTc2MDkyMzN9.4d2qHwN6_I9H-FavmGDz7Eu9SWRXAnxBH0W7PAG9OTo" 127.0.0.1:5000/api/v1/users/
{
  "code": 200,
  "data": {
    "age": "18",
    "cache": true,
    "id": "2",
    "mobile": "13200001111",
    "username": "12345678"
  },
  "message": "SUCCESS",
  "state": 0
}

# 刷新token
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYwOTExMywianRpIjoiNTBmNGM2NTUtYWNkNC00ZjAxLTg3NWItNTYwYjIxY2VlMTY5IiwidHlwZSI6InJlZnJlc2giLCJzdWIiOjIsIm5iZiI6MTY5NzYwOTExMywiZXhwIjoxNjk3NjA5NDEzfQ.PnqLxzlDEgzwlrTNhsBE7xzTPGUR8jOaceRHiMEA-Y0" -X POST 127.0.0.1:5000/api/v1/refresh
{
  "code": 200,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYwOTE4MCwianRpIjoiNjY5NDU2MGQtMDY3NS00NDZiLWE1YWQtODExYWNlODc3MzYyIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6MiwibmJmIjoxNjk3NjA5MTgwLCJleHAiOjE2OTc2MDkzMDB9.jxrgHVh4KmPVNyHLxxMUY2MIB719KXDxHv3S9UFUBy4"
  },
  "message": "SUCCESS",
  "state": 0
}
```