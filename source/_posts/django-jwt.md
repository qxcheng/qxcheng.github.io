---
categories:
- - django
date: 2024-11-12 15:15:25
description: 本文将详细介绍如何使用 `djangorestframework-simplejwt` 库为自定义用户模型实现 JWT 认证与授权功能。我们将从配置文件开始，逐步讲解如何将该库集成到
  Django 项目中，特别是在使用自定义用户模型的情况下，确保安全、灵活的用户认证系统。
id: '330'
tags:
- web框架
title: Django入门系列5-实现JWT认证功能
---


本文将详细介绍如何使用 `djangorestframework-simplejwt` 库为自定义用户模型实现 JWT 认证与授权功能。我们将从配置文件开始，逐步讲解如何将该库集成到 Django 项目中，特别是在使用自定义用户模型的情况下，确保安全、灵活的用户认证系统。

## 1.myproject/settings.py

设置文件中的配置：

*   安装的应用中添加simplejwt
*   设置默认使用自定义的jwt认证方案
*   设置REFRESH\_TOKEN的有效期

```python
INSTALLED_APPS = [
    'rest_framework_simplejwt',
]

# 使用jwt认证作为后台认证方案
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'myapp.authentication.MyJWTAuthentication',
    ],
}

# 设置REFRESH_TOKEN的有效期
SIMPLE_JWT = {
    'REFRESH_TOKEN_LIFETIME': timedelta(days=30),
    'ROTATE_REFRESH_TOKENS': True,
}
```

## 2.myapp/models.py

在用户模型中添加 `password` 和 `is_active` 字段，及 `check_password` 方法。

```python
class Author(models.Model):
    ...
    password = models.CharField(max_length=20)
    is_active = models.BooleanField(default=True)

    def check_password(self, password):
        return password == self.password
```

## 3.myapp/views.py

我们需要新建一个 `LoginView` 来提供登录视图，根据前端提供的用户名密码，验证通过后生成 access token和 refresh token，并返回。对于不需要JWT鉴权的接口可以使用 `authentication_classes` 及 `permission_classes` 来说明。 另外修改 `ArticleList` 的 get 方法，使用从 token 中解析出的user对象来获取其所有文章。

```python
from rest_framework.permissions import AllowAny

from .serializers import LoginSerializer


class LoginView(APIView):
    authentication_classes = []  
    permission_classes = [AllowAny]
   
    def post(self, request, *args, **kwargs):
        serializer = LoginSerializer(data=request.data, context={'request': request})
        serializer.is_valid(raise_exception=True)
        refresh = serializer.context.get('refresh')
        access = serializer.context.get('access')
        return Response({'code': 200, 'msg': '登录成功', 'refresh': refresh, 'access': access})


class ArticleList(APIView):

    # 获取所有文章
    def get(self, request, format=None):
        articles = Article.objects.filter(author=request.user.id)
        serializer = ArticleSerializer(articles, many=True)
        return Response(serializer.data)
```

## 4.myapp/serializers.py

主要通过 `LoginSerializer` 完成验证用户名密码并生成token的过程。

```python
from rest_framework.exceptions import ValidationError
from rest_framework_simplejwt.tokens import RefreshToken


class LoginSerializer(serializers.ModelSerializer):
    name = serializers.CharField()

    class Meta:
        model = Author
        fields = ('name', 'password')

    def validate(self, attrs):
        username = attrs.get('name')
        password = attrs.get('password')

        user = Author.objects.get(name=username)
        if user and user.check_password(password):
            refresh = RefreshToken.for_user(user)
            self.context['refresh'] = str(refresh)
            self.context['access'] = str(refresh.access_token)
        else:
            raise ValidationError({'user': 'login error'})

        return attrs
        
class AuthorSerializer(serializers.ModelSerializer):    
    ...
    password = serializers.CharField(max_length=20, required=True)
```

## 5.myproject/authentication.py

通过继承 `JWTAuthentication` 来自定义其他需要JWT鉴权的接口的token解析认证过程。

```python
from rest_framework.exceptions import AuthenticationFailed
from rest_framework_simplejwt.authentication import JWTAuthentication

from .models import Author


class MyJWTAuthentication(JWTAuthentication):

    def authenticate(self, request):
        jwt_value = self.get_raw_token(self.get_header(request))
        if not jwt_value:
            raise AuthenticationFailed('token 字段是必须的')

        validated_token = self.get_validated_token(jwt_value)
        user = Author.objects.get(pk=validated_token['user_id'])
        return user, jwt_value
```

## 6.myproject/urls.py

提供获取和刷新token的url地址，这两个url分别对应 `MyObtainTokenPairView` 和 `TokenRefreshView` 两个视图。

```python
from rest_framework_simplejwt.views import TokenRefreshView

from myapp.views import LoginView

urlpatterns = [
 ...
    path('api/v1/login/', LoginView.as_view(), name='token_obtain'),
    path('api/v1/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
]
```

## 7.测试

```bash
# docker-compose重新构建web
docker-compose up -d --build web

# 数据库迁移
docker exec -it django-tutorial-web-1 /bin/sh
python manage.py makemigrations myapp
python manage.py migrate

# 获取token
curl -H "Content-Type: application/json" -X POST -d '{"name":"author108", "password":"123456"}' 127.0.0.1:5000/api/v1/login/

{"code":200,"msg":"登录成功","refresh":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTczMzExMDg3MiwiaWF0IjoxNzMwNTE4ODcyLCJqdGkiOiI5NjI1NmM0ZmVjNjY0Y2MzOWQ3Mjc3OGM4YmM0MTZiZCIsInVzZXJfaWQiOjF9.IfzLFiGuqChYMBCAFgjWOHb2fuyRIkhoZ5CvNCJ35BA","access":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzMwNTE5MTcyLCJpYXQiOjE3MzA1MTg4NzIsImp0aSI6IjkyNDg3MzJlYmI0ODRhYTQ5ODJhMzZmMTFlOGJhZTMwIiwidXNlcl9pZCI6MX0.6W6ZhvGERpwTsh6AHPx5By5eAHDZ5lsPuXA1WOyZ8ws","user_id":null}

# 刷新token
curl -H "Content-Type: application/json" -X POST -d '{"refresh":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTczMzExMDg3MiwiaWF0IjoxNzMwNTE4ODcyLCJqdGkiOiI5NjI1NmM0ZmVjNjY0Y2MzOWQ3Mjc3OGM4YmM0MTZiZCIsInVzZXJfaWQiOjF9.IfzLFiGuqChYMBCAFgjWOHb2fuyRIkhoZ5CvNCJ35BA"}' 127.0.0.1:5000/api/v1/refresh/

{"access":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzMwNTE5NTY0LCJpYXQiOjE3MzA1MTg4NzIsImp0aSI6ImE4YmNiNDQzMjMzZDQwNmM4ZGRkODU3N2UxMjg2ZDI5IiwidXNlcl9pZCI6MX0.nQYz0MuMezoPbemz3hoyuewEOkwM32v8Mw2tK7_O2QE","refresh":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTczMzExMTI2NCwiaWF0IjoxNzMwNTE5MjY0LCJqdGkiOiJiOGVlNDE3NzZjZDM0ZWVlOTlkZGM2YTFjZjQ5YzQxYyIsInVzZXJfaWQiOjF9.t_FBq8Ro8ecxw9-QGMnHSKVkNCvC32TelpMmHrUF7j8"}

# 查询鉴权用户的所有文章
curl -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzMwNTE5NTY0LCJpYXQiOjE3MzA1MTg4NzIsImp0aSI6ImE4YmNiNDQzMjMzZDQwNmM4ZGRkODU3N2UxMjg2ZDI5IiwidXNlcl9pZCI6MX0.nQYz0MuMezoPbemz3hoyuewEOkwM32v8Mw2tK7_O2QE" -X GET 127.0.0.1:5000/api/v1/articles/

[{"id":1,"title":"mytitle","body":"mybody","create_date":"2024-11-02 11:46:00","author":{"id":1,"name":"author108","mobile":"108","age":0,"birth_date":"2001-01-01","mod_date":"2024-10-31 22:54:11","password":"123456","author_articles":[1],"is_active":true},"article_type":"短篇"}]
```