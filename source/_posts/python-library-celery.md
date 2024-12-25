---
categories:
- - Python
date: 2024-04-17 13:59:22
description: 'Python常用库7-celery'
id: '124'
tags:
- python
- 消息队列
title: Python常用库7-celery
---


## 1.普通任务、延迟任务

### tasks.py

```python
# tasks.py
import time
from celery import Celery

app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task(time_limit=60)  # 设置超时时间
def mytask(params):
    start = time.time()
    print('task start...')
    time.sleep(params['num'])
    print("task use：%s seconds " % (time.time() - start))

# 启动 Worker，监听 Broker 中是否有任务
# celery -A tasks worker --loglevel=info
```

### main.py

```python
# main.py
from tasks import mytask
import time 
from datetime import datetime, timedelta

def eta_second(second):
    ctime = datetime.now()
    utc_ctime = datetime.utcfromtimestamp(ctime.timestamp())
    time_delay = timedelta(seconds=second)  
    return utc_ctime + time_delay

def func(num):
    start = time.time()
    # 立即执行
    t = mytask.delay({"num": num})
    print("任务：%s 耗时：%s 秒 " % (t.task_id, time.time()-start))

def delay_func(num):
    start = time.time()
    # 延迟5s执行
    t = mytask.apply_async(args=({"num": num},), eta=eta_second(5))
    print("任务：%s 耗时：%s 秒 " % (t.task_id, time.time()-start))

if __name__ == '__main__':
    for i in range(3):
        func(i)        
        delay_func(i)  

# 启动传递任务
# python main.py
```

## 2.周期任务

### tasks.py

```python
# tasks.py
from celery import Celery
import celeryconfig

app = Celery('tasks', broker='redis://localhost:6379/0')
app.config_from_object('celeryconfig')

@app.task
def add(x, y):
    print(f'result: {x+y}')
    return x + y
```

### celeryconfig.py

```Python
# celeryconfig.py
from celery.schedules import crontab

CELERYBEAT_SCHEDULE={
    "every-1-minute": {
        'task': 'tasks.add',
        'schedule': crontab(minute='*/1'),  # 表示一分钟执行一次
        'args': (5, 6)                      # 传入的参数
    }
}

# 启动
celery -A tasks worker -B --loglevel=info   # -s /tmp/celerybeat-schedule 
```