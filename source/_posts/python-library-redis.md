---
categories:
- - Python
date: 2024-04-17 13:53:13
description: 'Python常用库5-redis'
id: '119'
tags:
- python
- 缓存
title: Python常用库5-redis
---


## 1.基本使用

```python
import redis

pool = redis.ConnectionPool(host='localhost', port=6379, decode_responses=True)  
r = redis.Redis(connection_pool=pool)

if __name__ == '__main__':
    # 字符串
    r.set('str1', 1, ex=3)  # 过期时间3s
    r.expire('str1', 10)    # 设置过期时间为10s
    if r.exists("str1"):    # 键是否存在
        v = r.get('str1')   # 获取值
        print(v, type(v))   # 1 <class 'str'>
        r.delete("str1")    # 删除键

    # hash
    r.hset("hash1", "k1", "v1")  # 设置键值对
    r.hset("hash1", "k2", "v2")
    v = r.hget("hash1", "k1")    # 指定键值对
    print(v, type(v))            # v1 <class 'str'>
    print(r.hgetall("hash1"))    # 所有键值对 {'k1': 'v1', 'k2': 'v2'}
    print(r.hlen("hash1"))       # 键值对个数 2
    print(r.hkeys("hash1"))      # 所有键 ['k1', 'k2']
    print(r.hvals("hash1"))      # 所有值 ['v1', 'v2']
    r.hdel("hash1", "k1")        # 删除键值对
    print(r.hexists("hash1", "k1"))  # 键是否存在 False

    # 列表
    r.lpush("list1", 1, 2, 3)     # 队头添加值
    r.rpush("list1", 4, 5, 6)     # 队尾添加值
    r.lrem('list1', 1, 2)         # 删除1个值为2的元素
    v = r.lrange("list1", 0, -1)  # 取出列表的值
    print(v)                      # ['3', '1', '4', '5', '6']
    print(r.llen("list1"))        # 列表长度 5

    # 集合
    r.sadd("set1", 33, 44, 55, 66)  # 添加到集合
    r.srem("set1", 33)              # 删除集合元素
    v = r.smembers("set1")          # 获取集合元素
    print(v)                        # {'55', '44', '66'}
    print(r.scard("set1"))          # 集合长度 3

    # 有序集合
    r.zadd("zset1", {'A': 100, 'B': 90, 'C': 80})  # 添加到集合
    r.zrem("zset1", 'A')            # 删除集合元素
    v = r.zrevrange('zset1', 0, 1)  # 返回前两名元素
    print(v)                        # ['B', 'C']

    # 计数器
    r.incr('counter')      # 自增1
    r.incr('counter')
    r.incr('counter', 2)   # 自增2
    r.decr('counter')      # 自减1
    v = r.get('counter')
    print(v)               # 3

    # 管道：一次性执行多条命令
    pipe = r.pipeline()      # 创建一个管道
    pipe.lpush("list2", 1)
    pipe.lpush("list2", 2)
    pipe.incr('num', 2)    
    result = pipe.execute()  # 一次执行多条命令
    print(result)            # [1, 2, 2]
```