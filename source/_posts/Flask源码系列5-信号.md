---
categories:
- - flask
date: 2024-04-19 10:16:26
description: "本篇文章介绍python中信号机制的使用。"
id: '176'
tags:
- python
- web框架
title: Flask源码系列5-信号
---


本篇文章介绍python中信号机制的使用。

flask源码：

```python
# signals.py
from blinker import Namespace

_signals = Namespace()

template_rendered = _signals.signal("template-rendered")  
before_render_template = _signals.signal("before-render-template")  
request_started = _signals.signal("request-started")
...

# app.py
from .signals import request_started

class Flask(App):
    ...
    def full_dispatch_request(self) -> Response:
        ...
        try:  
            request_started.send(self, _async_wrapper=self.ensure_sync)
            ...
```

## 1.原理

Blinker是用于在Python对象之间创建灵活的信号（事件）机制的Python库。

1.  **信号（Signal）**：Blinker通过创建“信号”对象来实现事件通知。这些信号可以被任意数量的“订阅者”监听。
    
2.  **订阅者（Subscriber）**：订阅者是连接到特定信号的函数。当信号被“发送”（即触发）时，所有订阅了该信号的函数都会被调用。
    
3.  **发送信号**：发送信号通常是指在代码的某个点触发信号，导致所有连接到该信号的订阅者函数被执行。
    
4.  **松耦合**：使用Blinker可以使应用程序的不同部分保持松耦合，因为它们不需要直接调用对方的代码。相反，它们通过信号和订阅者机制进行交互。
    

## 2.flask使用

```python
from flask import Flask
from flask.signals import request_started

app = Flask(__name__)

# 信号处理函数
def before_request(sender, **extra):
    print("Before request started")

# 连接信号
request_started.connect(before_request, app)

@app.route('/')
def index():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run()
```

## 3.blinker使用

**基础使用：**

```python
from blinker import signal

# 创建信号
some_signal = signal('my_signal')

# 订阅者函数
def subscriber(sender, **kwargs):
    print("Got a signal from:", sender)
    print("With data:", kwargs)

# 连接订阅者到信号
some_signal.connect(subscriber)

# 发送信号
some_signal.send('some_sender', data="Some data")

$
Got a signal from: some_sender
With data: {'data': 'Some data'}
```

**使用装饰器：**

```python
@some_signal.connect
def decorated_subscriber(sender, **kwargs):
    print("Received signal from decorated subscriber:", sender)
```

**使用类：**

```python
class Processor:
    def __init__(self):
        some_signal.connect(self.process)

    def process(self, sender, **kwargs):
        print(f"Processed signal in class from: {sender}")

processor = Processor()
some_signal.send('class_sender')
```

在这个例子中，`Processor` 类的实例在初始化时订阅了信号。当信号被发送时，它的 `process` 方法被调用。