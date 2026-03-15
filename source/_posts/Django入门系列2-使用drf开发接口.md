---
categories:
- - django
date: 2024-11-12 15:09:55
description: 在本篇文章中，我们将详细介绍如何使用 **Django REST Framework**（简称 DRF）开发一个简单的 API 接口。我们将从创建数据库、定义模型、编写序列化器，到实现视图函数以及路由配置，逐步展开，带你了解完整的开发流程。无论你是刚接触
  Django 的开发者，还是希望通过 DRF 快速构建 RESTful API 的开发者，都能从中获得清晰的思路和实用的技巧。
id: '323'
tags:
- web框架
title: Django入门系列2-使用drf开发接口
---


在本篇文章中，我们将详细介绍如何使用 **Django REST Framework**（简称 DRF）开发一个简单的 API 接口。我们将从创建数据库、定义模型、编写序列化器，到实现视图函数以及路由配置，逐步展开，带你了解完整的开发流程。无论你是刚接触 Django 的开发者，还是希望通过 DRF 快速构建 RESTful API 的开发者，都能从中获得清晰的思路和实用的技巧。

## 1.文件结构

如下是本篇完成后的项目文件结构：

```python
.
├── data
│   ├── mysql
│   └── redis
├── docker-compose.yml
├── Dockerfile
├── myproject
│   ├── manage.py
│   ├── myapp
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── tests.py
│   │   ├── urls.py
│   │   └── views.py
│   └── myproject
│       ├── asgi.py
│       ├── __init__.py
│       ├── settings.py
│       ├── urls.py
│       └── wsgi.py
└── requirements.txt
```

## 2.创建数据库

首先进入mysql容器创建一个数据库：

```bash
docker exec -it django-tutorial-mysql-1 /bin/sh

mysql -u root -p
root

CREATE DATABASE django_demo default charset=utf8mb4;
```

## 3.docker-compose.yml

在docker-compose中进行数据库迁移，然后启动服务：

```yml
services:
  web:
    build: .
    privileged: true
    command:
      - sh
      - -c
      - 
          python manage.py migrate
          python manage.py runserver 0.0.0.0:5000
    volumes:
      - ./data/myproject/myapp/migrations:/code/myproject/myapp/migrations
...
```

## 4.Dockerfile

在Dockerfile中注释掉启动命令：

```yml
#CMD ["python", "manage.py", "runserver", "0.0.0.0:5000"]
```

## 5.myproject/settings.py

在配置文件中配置数据库连接：

```python
import pymysql
pymysql.install_as_MySQLdb()

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'django_demo',   # 数据库名称 数据库需要提前建好
        'USER': 'root',          # 数据库用户名
        'PASSWORD': 'root',      # 数据库密码
        'HOST': 'mysql',         # 数据库主机
        'PORT': 3306,            # 数据库端口
        'CONN_MAX_AGE': 10,      # 数据库连接生命周期（秒），默认0为即用即丢，None为永久保持连接
        'OPTIONS': {
            'init_command': 'SET default_storage_engine=INNODB;',
            'charset': 'utf8mb4'
        }
    }
}
```

## 6.myapp/models.py

此处定义数据库模型，以通过 Django 的 ORM 操作数据库。

```python
from django.db import models


class Article(models.Model):

    title = models.CharField(max_length=90, db_index=True)
    body = models.TextField(blank=True)
    create_date = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-create_date']
        verbose_name = "Article"
        verbose_name_plural = "Articles"
```

## 7.myapp/serializers.py

此处编写数据序列化代码，其主要用于Django模型实例->Python数据类型->前端JSON等格式的数据转换，或反向转换，并提供了强大的数据验证和自定义功能。

```python
from rest_framework import serializers

from .models import Article


class ArticleSerializer(serializers.ModelSerializer):

    title = serializers.CharField(max_length=90, required=True)
    body = serializers.CharField(max_length=2048, required=True)
    create_date = serializers.DateTimeField(format="%Y-%m-%d %H:%M:%S", required=False, read_only=True)

    class Meta:
        model = Article
        fields = '__all__'
```

## 8.myapp/views.py

此处编写视图代码，通过 API 视图（`APIView`）来实现 GET、POST、PUT 和 DELETE 请求处理，实现基本的增删改查（CRUD）操作。

```python
from rest_framework import status
from rest_framework.views import APIView
from rest_framework.response import Response
from django.http import Http404

from .serializers import ArticleSerializer
from .models import Article


class ArticleList(APIView):
 # 获取文章列表
    def get(self, request, format=None):
     # 从数据库查数据
        articles = Article.objects.all()
        # 序列化数据库中的数据
        serializer = ArticleSerializer(articles, many=True)
        # 返回给客户端
        return Response(serializer.data)

 # 新增一篇文章
    def post(self, request, format=None):
     # 序列化客户端发送的数据
        serializer = ArticleSerializer(data=request.data)
        # 进行数据校验
        if serializer.is_valid():
         # 保存到数据库
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    
class ArticleDetail(APIView):
    def get_object(self, pk):
        try:
            return Article.objects.get(pk=pk)
        except Article.DoesNotExist:
            raise Http404

 # 获取一篇文章
 # pk是在路由中指定的参数
    def get(self, request, pk, format=None):
        article = self.get_object(pk)
        serializer = ArticleSerializer(article)
        return Response(serializer.data)

 # 修改一篇文章
    def put(self, request, pk, format=None):
        article = self.get_object(pk)
        serializer = ArticleSerializer(instance=article, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

 # 删除一篇文章
    def delete(self, request, pk, format=None):
        article = self.get_object(pk)
        article.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

## 9.myapp/urls.py

此处为视图函数编写路由代码，将视图与 URL 路由关联，便于客户端请求访问。

```python
from django.urls import re_path
from rest_framework.urlpatterns import format_suffix_patterns

from . import views

urlpatterns = [
    re_path(r'^articles/$', views.ArticleList.as_view()),
    re_path(r'^articles/(?P<pk>[0-9]+)$', views.ArticleDetail.as_view()),
]

urlpatterns = format_suffix_patterns(urlpatterns)
```

## 10.myproject/urls.py

此处引用路由文件，可以添加路由前缀：

```python
from django.urls import path, include

urlpatterns = [
 ...
    path('api/v1/', include('myapp.urls')),
]
```

## 11.测试

通过 `curl` 或其他 HTTP 客户端工具对 API 进行测试，验证接口的正确性。

```python
# docker-compose构建web
docker-compose up -d --build web

# 生成迁移文件
# 注：一般在测试环境中完成生成makemigrations文件的步骤，生产环境只进行migrate
docker exec -it django-tutorial-web-1 /bin/sh
python manage.py makemigrations myapp

# docker-compose重启web
docker-compose restart web 

# 增
curl -H "Content-Type: application/json" -X POST -d '{"title":"mytitle", "body":"mybody"}' 127.0.0.1:5000/api/v1/articles/
{"id":1,"title":"mytitle","body":"mybody","create_date":"2023-10-23 09:17:35"}

# 查列表
curl 127.0.0.1:5000/api/v1/articles/
[{"id":1,"title":"mytitle","body":"mybody","create_date":"2023-10-23 09:17:35"}]

# 查
curl 127.0.0.1:5000/api/v1/articles/1
{"id":1,"title":"mytitle","body":"mybody","create_date":"2023-10-23 09:17:35"}

# 改
curl -H "Content-Type: application/json" -X PUT -d '{"title":"notitle", "body":"nobody"}' 127.0.0.1:5000/api/v1/articles/1
{"id":1,"title":"notitle","body":"nobody","create_date":"2023-10-23 09:17:35"}

# 删
curl -X DELETE 127.0.0.1:5000/api/v1/articles/1
```