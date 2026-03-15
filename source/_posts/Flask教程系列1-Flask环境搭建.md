---
categories:
- - flask
date: 2024-04-19 09:29:58
description: 本系列文章主要介绍Python的一款轻量化web框架flask的使用，这里使用docker-compose来快速搭建开发环境。由于本人经验有限，文章难免有纰漏之处，还请海涵。
id: '144'
tags:
- python
- web框架
title: Flask教程系列1-Flask环境搭建
---


本系列文章主要介绍Python的一款轻量化web框架flask的使用，这里使用docker-compose来快速搭建开发环境。由于本人经验有限，文章难免有纰漏之处，还请海涵。

## 1.安装docker

```bash
# 安装必要的软件
sudo apt install apt-transport-https ca-certificates curl software-properties-common gnupg lsb-release

# 添加软件源的GPG密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg  sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 添加 Docker 软件源
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## 2.安装docker-compose

```bash
curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

## 3.项目文件结构

如下是本篇完成后的项目文件结构：

```bash
.
├── data
│   ├── mysql
│   └── redis
├── docker-compose.yml
├── Dockerfile
├── requirements.txt
└── wsgi.py
```

## 4.docker-compose.yml

这里共起了3个服务，分别是web、mysql、redis，将mysql和redis的数据挂载到了宿主机，服务之间通过新建的桥接网络通信。

```yml
version: '2'

networks:
  backend:
    driver: bridge

services:
  web:
    build: .
    privileged: true
    ports:
      - "5000:5000"
    networks:
      - backend
    depends_on:
      - mysql
      - redis

  redis:
    image: redis:5.0
    environment:
      - TZ=Asia/Shanghai
    privileged: true
    volumes:
      - ./data/redis:/data 
    ports:
      - "6379:6379" 
    networks:
      - backend
    restart: always

  mysql:
    image: mysql:5.7
    environment:
      - TZ=Asia/Shanghai
      - MYSQL_USER=admin 
      - MYSQL_PASSWORD=admin
      - MYSQL_ROOT_PASSWORD=root
    privileged: true
    volumes:
      - ./data/mysql:/var/lib/mysql 
    ports:
      - "3306:3306"  
    networks:
      - backend
    restart: always
```

## 5.Dockerfile

这里是web服务的Dockerfile，首先设置了pip源，然后拷贝requirements.txt安装依赖包，最后拷贝项目文件并启动flask自带的服务。

```yml
FROM python:3.8-alpine

RUN pip config set global.index-url http://mirrors.aliyun.com/pypi/simple
RUN pip config set install.trusted-host mirrors.aliyun.com

WORKDIR /code
COPY requirements.txt /code/requirements.txt
RUN pip install -r requirements.txt

ADD . /code
CMD ["python", "wsgi.py"]
```

## 6.requirements.txt

```text
Flask==3.0.0
```

## 7.wsgi.py

flask项目的hello world，后面会进行扩展。

```python
from flask import Flask

app = Flask(__name__)

@app.route('/', methods=['GET', 'POST'])            
def hello_world(): 
    return 'hello_world' 

if __name__ == '__main__':
    app.run('0.0.0.0', 5000)
```

## 8.测试

我们在两个命令行窗口中分别起服务和发送http请求，预期能打印出hello\_world。

```bash
# bash1
docker-compose up --build

# bash2
curl 127.0.0.1:5000
hello_world
```