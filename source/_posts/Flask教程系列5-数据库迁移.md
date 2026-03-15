---
categories:
- - flask
date: 2024-04-19 09:45:21
description: 本篇文章我们介绍在更改表结构后，如何对数据库进行迁移。
id: '153'
tags:
- python
- web框架
title: Flask教程系列5-数据库迁移
---


本篇文章我们介绍在更改表结构后，如何对数据库进行迁移。

## 1.docker-compose.yml

首先挂载我们的迁移文件夹，以对迁移版本信息进行持久化。

```yml
services:
  web:
    build: .
    privileged: true
    volumes:
      - ./data/migrations:/code/migrations
    ports:
      - "5000:5000"
    ...
```

## 2.apps/models/user.py

我们分别对user表进行了改、增、删字段的处理：

```python
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    mobile = db.Column(EncType)
    username = db.Column(db.String(200))  # 100 -> 200  修改
    password = db.Column(db.String(120))
    age = db.Column(db.Integer)
    birth = db.Column(db.DateTime)
    photo = db.Column(db.String(120))     # 新增

    create_time = db.Column(db.DateTime, default=datetime.now)
    update_time = db.Column(db.DateTime, default=datetime.now)
    #delete_time = db.Column(db.DateTime, default=datetime.now)  删除

    __table_args__ = (
        db.Index("index_mobile", mobile, mysql_length=16),
    )
```

## 3.执行迁移

我们需要进入flask容器执行迁移命令：

```bash
# 进入flask容器
docker exec -it flasktutorial_web_1 /bin/sh

# 迁移初始化，只需执行一次
flask db init

# 执行迁移
flask db migrate -m '2023-08-24 17:00 migrate'
flask db upgrade

## 其他命令
# 执行回滚
flask db downgrade 

# 之前低版本的库，如果字段类型、大小的迁移没有成功，需要
# 在 migrations/env.py 文件的 run_migrations_online 函数添加如下内容：
context.configure(
      …………
      compare_type=True,           # 检查字段类型
      compare_server_default=True, # 比较默认值
      )
```

## 4.测试

我们进入mysql容器查看表结构，检查已经迁移成功。

```msyql
mysql> use flask-demo;
mysql> describe user;
+-------------+--------------+------+-----+---------+----------------+
 Field        Type          Null  Key  Default  Extra          
+-------------+--------------+------+-----+---------+----------------+
 id           int(11)       NO    PRI  NULL     auto_increment 
 mobile       blob          YES   MUL  NULL                    
 username     varchar(200)  YES        NULL                    
 password     varchar(120)  YES        NULL                    
 age          int(11)       YES        NULL                    
 birth        datetime      YES        NULL                    
 create_time  datetime      YES        NULL                    
 update_time  datetime      YES        NULL                    
 photo        varchar(120)  YES        NULL                    
+-------------+--------------+------+-----+---------+----------------+
```