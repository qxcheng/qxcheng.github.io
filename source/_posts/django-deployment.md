---
categories:
- - django
date: 2024-11-12 15:19:30
description: Django 部署是将开发环境迁移到生产环境的一项关键任务。本文将介绍如何使用 Docker、Nginx 和 Gunicorn 来部署 Django
  应用。通过这三者的配合，我们可以实现高效、可扩展和易于维护的生产环境。
id: '339'
tags:
- web框架
title: Django入门系列8-项目部署
---


Django 部署是将开发环境迁移到生产环境的一项关键任务。本文将介绍如何使用 Docker、Nginx 和 Gunicorn 来部署 Django 应用。通过这三者的配合，我们可以实现高效、可扩展和易于维护的生产环境。

## 1.部署架构概述

在这个部署方案中，我们使用以下技术栈：

*   **Docker**：用于容器化 Django 应用及其依赖，确保在不同环境中一致运行。
*   **Gunicorn**：作为 WSGI 服务器，负责运行 Django 应用，处理来自 Nginx 的请求。
*   **Nginx**：作为反向代理服务器，将客户端的请求转发到 Gunicorn，并处理静态文件和负载均衡。

## 2.项目文件结构

如下是我们项目的最终文件结构：

```python
├── data
│   ├── myproject
│   │   └── myapp
│   │       └── migrations
│   ├── mysql
│   ├── nginx
│   │   └── nginx.conf
│   └── redis
├── myproject
│   ├── gunicorn.py
│   ├── manage.py
│   ├── myapp
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── authentication.py
│   │   ├── middleware.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── signals.py
│   │   ├── tests.py
│   │   ├── urls.py
│   │   └── views.py
│   ├── myproject
│   │   ├── __init__.py
│   │   ├── asgi.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
├── Dockerfile
├── docker-compose.yml
├── .dockerignore
└── requirements.txt
```

## 3.gunicorn.py

Gunicorn 是 Python 的一个 WSGI 服务器，它通过监听指定的端口并转发请求给 Django 应用。在django项目根目录下创建 `gunicorn.py` 文件，配置 Gunicorn 服务器，其监听 5000 端口：

```python
import os
import gevent.monkey

gevent.monkey.patch_all()

# 创建日志文件夹
logdir = './logs'
if not os.path.exists(logdir):
    os.mkdir(logdir)

# 日志
loglevel = 'warning'
current_path = os.path.dirname(__file__)
errorlog = os.path.join(current_path, logdir, 'gunicorn-error.log')
accesslog = os.path.join(current_path, logdir, 'gunicorn-access.log')
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" "%(D)s"'

# 工作进程
worker_class = 'gevent'
workers = os.cpu_count() * 2 + 1
threads = os.cpu_count() * 2 + 1

timeout = 60

# 指定监听地址和端口
bind = "0.0.0.0:5000"
```

## 4.nginx.conf

Nginx 作为反向代理，接收客户端的请求并将请求转发到 Gunicorn。在`data/nginx`目录下创建一个名为 `nginx.conf` 的文件，配置 Nginx 作为反向代理，其监听80端口，并将所有请求转发到Gunicorn的5000端口：

```python
user  nginx;  
  
events {  
    worker_connections 5000;  
}  

http {  
 server {  
  listen 80;  
  location / {  
   proxy_pass http://web:5000;  
  }  
 }  
}
```

## 5.docker-compose.yml

1.  修改django项目的启动方式为gunicorn
2.  添加nginx容器，其将宿主机的8080端口映射到容器80端口

```python
services:
  web:
    ...
    command:
      - sh
      - -c
      - 
          python manage.py migrate
          gunicorn -c gunicorn.py myproject.wsgi:application
  
  nginx:
    image: nginx:alpine
    volumes:
      - ./data/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    ports:
      - "8080:80"
    depends_on:
      - web
    networks:
      - backend
```

## 6.myproject/settings.py

在配置文件中添加 `ALLOWED_HOSTS`

```python
ALLOWED_HOSTS = ['web']
```

## 7.测试

```python
# docker-compose重新构建
docker-compose up -d --build

# 获取一篇文章，改用8080端口
curl -X GET 127.0.0.1:8080/api/v1/articles/1

{"id":1,"title":"mytitle","body":"mybody++","create_date":"2024-10-25 00:00:00","author":{"id":10,"name":"author107","mobile":"107","age":0,"birth_date":"2001-01-01","mod_date":"2024-10-25 15:15:36","password":"123456","author_articles":[1],"is_active":true},"article_type":"短篇"}
```