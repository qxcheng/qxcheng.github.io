---
categories:
- - flask
date: 2024-04-19 09:46:34
description: 本篇文章简单介绍redis在flask中的使用。
id: '155'
tags:
- python
- web框架
title: Flask教程系列6-使用缓存
---


本篇文章简单介绍redis在flask中的使用。

## 1.apps/api\_v1/user.py

我们在user视图代码中，通过导入对象g，从而使用通过钩子函数关联的redis连接对象。使用缓存的大致思路是，对于put更新、delete删除请求需要同时删除缓存，对于post新增请求可以添加缓存。对于get请求，优先查询缓存，未命中时查询数据库后添加缓存，使得下次查询可以命中缓存。

```python
from typing import Union
from flask.views import MethodView
from flask import request, g

from apps.api_v1 import api_v1
from apps.forms.user import SignUpForm
from apps.utils.response import HttpResponse
from apps.extensions import db
from apps.models.user import User

class UserAPI(MethodView):
    USER_CACHR_PREFIX = "USERID::"

    def extract_user_info(self, user: User) -> dict:
        return {
            "username": user.username,
            "id": user.id,
            "age": user.age,
            "mobile": user.mobile
        }

    def get_user_info(self, user: Union[User,int]) -> dict:
        if isinstance(user, User):
            result = self.extract_user_info(user)
            self.add_cache(result)
            return result
        elif isinstance(user, int):
            result = g.redis.hgetall(f'{self.USER_CACHR_PREFIX}{user}')
            if result:
                result["cache"] = True
                return result
            else:
                user = User.query.get(user)
                if not user:
                    return {}
                else:
                    result = self.extract_user_info(user)
                    self.add_cache(result)
                    return result

    def add_cache(self, user_info: dict):
        g.redis.hmset(f'{self.USER_CACHR_PREFIX}{user_info["id"]}', user_info)

    def delete_cache(self, user_id: int):
        g.redis.delete(f'{self.USER_CACHR_PREFIX}{user_id}')

    def user_not_found(self, user_id: int) -> str:
        return f"user not found: {user_id}"

    def get(self, user_id: int):
        # get users or get user
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
            return HttpResponse.success(data=self.get_user_info(user))
        else:
            message = f"add user fail:{form.errors}"
            return HttpResponse.params_error(message=message)

    def delete(self, user_id: int):
        # delete user
        user = User.query.get(user_id)
        if not user:
            return HttpResponse.params_error(message=self.user_not_found(user_id))
        else:
            with db.auto_commit():
                db.session.delete(user)
            self.delete_cache(user.id)
            return HttpResponse.success(message=f'delete user success: {user_id}')

    def put(self, user_id: int):
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

            self.delete_cache(user.id)

            return HttpResponse.success(data=self.get_user_info(user))
        else:
            message = f"update user fail:{form.errors}"
            return HttpResponse.params_error(message=message)

user_view = UserAPI.as_view('user_api')
api_v1.add_url_rule('/users/', defaults={'user_id': None}, view_func=user_view, methods=['GET',])
api_v1.add_url_rule('/users/', view_func=user_view, methods=['POST',])
api_v1.add_url_rule('/users/<int:user_id>', view_func=user_view, methods=['GET', 'PUT', 'DELETE'])   
```

## 2.测试

通过data中的cache字段，我们可以判断使用了缓存。

```python
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
curl 127.0.0.1:5000/api/v1/users/1
{
  "code": 200,
  "data": {
    "age": "18",
    "cache": true,
    "id": "1",
    "mobile": "13200001111",
    "username": "12345678"
  },
  "message": "SUCCESS",
  "state": 0
}
```