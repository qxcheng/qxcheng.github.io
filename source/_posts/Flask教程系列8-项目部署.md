---
categories:
- - flask
date: 2024-04-19 09:49:29
description: 本篇文章介绍使用gunicorn以多进程+多线程的方式来部署flask项目，并使用了nginx做转发。
id: '159'
tags:
- python
- web框架
title: Flask教程系列8-项目部署
---


本篇文章介绍使用gunicorn以多进程+多线程的方式来部署flask项目，并使用了nginx做转发。

## 1.文件结构

如下是我们最终的项目结构：

```python
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
│   ├── migrations
│   ├── mysql
│   ├── nginx
│   │   └── nginx.conf
│   └── redis
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
├── gunicorn.py
└── wsgi.py
```

## 2.docker-compose.yml

新增了nginx镜像，将宿主机的8080端口映射到容器的80端口，并挂载了nginx配置文件。

```python
services:
  web:
    ...
    expose:
      - "5000"

  nginx:
    image: nginx:latest
    volumes:
      - ./data/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    ports:
      - "8080:80"
    networks:
      - backend
```

## 3.data/nginx/nginx.conf

nginx配置将根路由转发到了flask的5000端口：

```yml
user  nginx;  

events {  
    worker_connections 5000;  
}  
http {  
    server {  
        listen 80;  
        location / {  
            proxy_pass http://web:5000;  
        }  
    }  
}
```

## 4.requirements.txt

```python
gunicorn==21.2.0
gevent==23.9.1
```

## 5.Dockerfile

dockerfile中改用gunicorn启动项目：

```python
#CMD ["python", "wsgi.py"]
ENTRYPOINT ["gunicorn", "--config", "gunicorn.py", "wsgi:app"]
```

## 6.gunicorn.py

gunicorn配置文件如下，其中指定了进程数和线程数，绑定了5000端口。

```python
import os
import gevent.monkey

gevent.monkey.patch_all()

# 创建日志文件夹
logdir = './logs'
if not os.path.exists(logdir):
    os.mkdir(logdir)

# 日志
loglevel = 'warning'
current_path = os.path.dirname(__file__)
errorlog = os.path.join(current_path, logdir, 'gunicorn-error.log')
accesslog = os.path.join(current_path, logdir, 'gunicorn-access.log')
access_log_format = '%(h)s %(l)s %(u)s %(t)s "%(r)s" %(s)s %(b)s "%(f)s" "%(a)s" "%(D)s"'

# 工作进程
worker_class = 'gevent'
workers = os.cpu_count() * 2 + 1
threads = os.cpu_count() * 2 + 1

timeout = 60
bind = "0.0.0.0:5000"
```

## 7.测试

使用gunicorn部署后测试，发现并发数差不多有两倍的提升。

```python
# 增 拿到一个token
curl -H "Content-Type: application/json" -X POST -d '{"username":"12345678", "password":"12345678", "mobile": "13200001111", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:5000/api/v1/users/

# 不使用gunicorn部署时的压测 -t线程数 -c连接数 -d测试时长
wrk -t10 -c1000 -d30s -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYwOTc0OCwianRpIjoiZDYxYTNmOGMtODNiNy00YTRmLWI5MmYtNWUwMTdkNGY3MDQxIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6MywibmJmIjoxNjk3NjA5NzQ4LCJleHAiOjE2OTc2MDk4Njh9.GG9fawbZEpkrTDmho9y5pc_ZprBpkz8UfuZwsW5lNbU" --latency "http://127.0.0.1:5000/api/v1/users/"

Running 30s test @ http://127.0.0.1:5000/api/v1/users/
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   329.79ms  323.15ms   2.00s    89.33%
    Req/Sec    70.78     50.50   252.00     61.68%
  Latency Distribution
     50%  220.38ms
     75%  234.83ms
     90%    1.20s 
     99%    1.41s 
  17933 requests in 30.10s, 5.92MB read
  Socket errors: connect 0, read 0, write 0, timeout 439
Requests/sec:    595.80
Transfer/sec:    201.32KB

# gunicorn部署后测试
curl -H "Content-Type: application/json" -X POST -d '{"username":"12345678", "password":"12345678", "mobile": "13200001111", "info":{"age": 18, "birth": "2023-08-23 12:30:59"}}' 127.0.0.1:8080/api/v1/users/

wrk -t10 -c1000 -d30s -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmcmVzaCI6ZmFsc2UsImlhdCI6MTY5NzYxMDAwNSwianRpIjoiZDA0ZDgwYWQtZjg0Zi00NzIwLWI0ZjUtY2E0NTRkZGMxOGFmIiwidHlwZSI6ImFjY2VzcyIsInN1YiI6NCwibmJmIjoxNjk3NjEwMDA1LCJleHAiOjE2OTc2MTAxMjV9.ZcjKQ6cd9JKAGBonT1FwEyaJCcZqdAP6yUPWCDj9ylc" --latency "http://127.0.0.1:8080/api/v1/users/"

Running 30s test @ http://127.0.0.1:8080/api/v1/users/
  10 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   225.38ms  320.71ms   1.99s    89.78%
    Req/Sec   112.12     74.17   700.00     76.77%
  Latency Distribution
     50%  117.37ms
     75%  120.03ms
     90%    1.11s 
     99%    1.32s 
  33331 requests in 30.06s, 9.06MB read
  Socket errors: connect 0, read 0, write 0, timeout 798
  Non-2xx or 3xx responses: 67
Requests/sec:   1108.68
Transfer/sec:    308.63KB
```