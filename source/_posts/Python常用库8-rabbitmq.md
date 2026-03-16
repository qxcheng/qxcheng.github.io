---
categories:
- - Python
date: 2024-04-17 14:04:09
description: 'Python常用库8-rabbitmq'
id: '126'
tags:
- python
- 消息队列
title: Python常用库8-rabbitmq
---

## 搭建

```Bash
sudo docker pull rabbitmq:3.9.9-management

# 最后面的为ImageID
sudo docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 -v `pwd`/data:/var/lib/rabbitmq/qxc --hostname my_rabbit -e RABBITMQ_DEFAULT_VHOST=my_vhost -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=adminxxx image_id

http://123.60.135.11:15672/

```

## 原理

`AMQP(Advanced Message Queuing Protocol)` 高级消息队列协议：高级消息队列协议。它是应用层协议的一个开放标准，为面向消息的中间件设计，基于此协议的客户端与消息中间件可传递消息，并不受产品、开发语言等条件的限制。

![](https://img2020.cnblogs.com/blog/1552936/202010/1552936-20201024103921637-693350551.png)

1、消息生产者连接到RabbitMQ Broker，创建connection，开启channel。

- Broker : 标识消息队列服务器实体rabbitmq-server
- Connection : 网络连接，比如一个TCP连接。
- Channel : 信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的TCP连接内的虚拟链接，AMQP命令都是通过信道发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说，建立和销毁TCP都是非常昂贵的开销，所以引入了信道的概念，以复用一条TCP连接。
- v-host : `Virtual Host` 虚拟主机，标识一批交换机、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个vhost本质上就是一个mini版的RabbitMQ服务器，拥有自己的队列、交换器、绑定和权限机制。vhost是AMQP概念的基础，必须在链接时指定，RabbitMQ默认的vhost是 /。

2、生产者声明交换机类型、名称、是否持久化等（其实这个说法不是很准确，其实大部分是在管理页面或者消费者端先声明的）。

3、生产者发送消息，并指定消息是否持久化等属性和routing key。

4、exchange收到消息之后，根据routing key路由到跟当前交换机绑定的相匹配的队列里面。

- Exchange: 交换机用来接收生产者发送的消息并将这些消息路由给服务器中的队列。
- Binding：exchange交换机和队列queue通过binging key绑定。一个绑定就是基于路由键将交换机和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。
- Exchange Type（路由不成功时会将消息抛弃）
    - fanout（路由到所有与它绑定的Queue中）
         不需要RoutingKey，只需要提前将Exchange与Queue进行绑定，一个Exchange可以绑定多个Queue，一个Queue可以同多个Exchange进行绑定。但是如果接受到消息的Exchange没有与任何Queue绑定，则消息会被抛弃。因为转发到全部绑定的队列上，所以广播模式存在重复消费的问题。 
    - direct（路由到binding key与routing key完全匹配的queue中）在管理后台将Exchange与Queue绑定，设置RoutingKey。（或在代码中绑定）
	- topic（路由到binding key与routing key模糊匹配的queue中）

任何发送到Topic Exchange的消息都会被转发到所有关心RouteKey中指定话题的Queue上。
每个队列都有其关心的主题，所有的消息都带有一个“标题”(RoutingKey路由键)，Exchange会将消息转发到所有关注主题能与RoutingKey模糊匹配的队列。主题模式需要RoutingKey，也需要提前绑定Exchange与Queue，在进行绑定时，要提供一个该队列关心的主题。Topic模式有两个关键词#和*。

“#”表示0个或若干个关键字，如“#.nanju.#”表示该队列关心所有涉及nanju的消息(一个RoutingKey为`asd.MQ.nanju.error`的消息会被转发到该队列)，

“_”表示一个关键字息，如“_.nanju.*”(一个RoutingKey为”`asd.nanju.error`”的消息会被转发到该队列，而”`asd.MQ.nanju.error`”则不会)

同样，如果Exchange没有发现能够与RoutingKey匹配的Queue，则会抛弃此消息。 - headers（根据消息的headers是否完全匹配Queue与Exchange绑定时指定的键值对）

5、消费者监听接收到消息之后开始业务处理，然后发送一个ack确认告知消息已经被消费（手动自动都有可能）。

- Queue : 消息队列，用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。

6、RabbitMQ Broker收到ack之后将对应的消息从队列里面删除掉。


## 1.封装基类

```python
# base.py
import json
import datetime
import time
import random
import pika
from pika.exceptions import ChannelClosed, ConnectionClosed

# rabbitmq 配置信息
MQ_CONFIG = {
    "hostname": "127.0.0.1",
    "port": 5672,
    "vhost": "my_vhost",
    "username": "admin",
    "password": "adminxxx",
    "exchange": "my_exchange",
    "queue": "my_queue",
    "routing_key": "my_key"
}

class RabbitMQServer(object):
    def __init__(self):
        self.config = MQ_CONFIG  # 配置文件加载
        self.host = self.config.get("hostname")
        self.port = self.config.get("port")
        self.username = self.config.get("username")
        self.password = self.config.get("password")
        self.vhost = self.config.get("vhost")
        self.exchange = self.config.get("exchange")
        self.queue = self.config.get("queue")
        self.routing_key = self.config.get("routing_key")

        self.connection = None
        self.channel = None

        # 关于队列的声明，如果使用同一套参数进行声明了，就不能再使用其他参数来声明
        self.arguments = {
            'x-message-ttl': 82800000,   # 设置队列中的所有消息的生存周期
            'x-expires': 82800000,       # 当队列在指定的时间没有被访问(consume, basicGet, queueDeclare…)就会被删除
            'x-max-length': 100000,      # 限定队列的消息的最大条数，超过指定条数将会把最早的几条删除掉
            'x-max-priority': 10         # 声明队列时先定义最大优先级值，在发布消息的时候指定该消息的优先级
        }

    def reconnect(self):
        try:
            if self.connection and not self.connection.is_closed:
                self.connection.close()

            credentials = pika.PlainCredentials(self.username, self.password)
            parameters = pika.ConnectionParameters(self.host, self.port, self.vhost, credentials)

            self.connection = pika.BlockingConnection(parameters)
            self.channel = self.connection.channel()
            self.channel.exchange_declare(exchange=self.exchange, exchange_type="direct", durable=True)
            self.channel.queue_declare(queue=self.queue, exclusive=False, durable=True, arguments=self.arguments) 
            self.channel.queue_bind(exchange=self.exchange, queue=self.queue, routing_key=self.routing_key)

            if isinstance(self, RabbitComsumer):
                self.channel.basic_qos(prefetch_count=1)  # prefetch 表明最大阻塞未ack的消息数量
                self.channel.basic_consume(on_message_callback=self.consumer_callback, queue=self.queue, auto_ack=False)

        except Exception as e:
            print("RECONNECT: ", e)

class RabbitPublisher(RabbitMQServer):
    def __init__(self):
        super(RabbitPublisher, self).__init__()

    def start_publish(self):
        self.reconnect()
        i = 1
        while True:
            message = {"value": i}
            try:
                self.channel.basic_publish(exchange=self.exchange, routing_key=self.routing_key, body=json.dumps(message))
                print("Publish value: ", i)
                i += 1
                time.sleep(3)
            except ConnectionClosed as e:
                print("ConnectionClosed: ", e)
                self.reconnect()
                time.sleep(2)
            except ChannelClosed as e:
                print("ChannelClosed: ", e)
                self.reconnect()
                time.sleep(2)
            except Exception as e:
                print("basic_publish: ", e)
                self.reconnect()
                time.sleep(2)

class RabbitComsumer(RabbitMQServer):
    def __init__(self):
        super(RabbitComsumer, self).__init__()

    def execute(self, body):
        body = body.decode('utf8')
        body = json.loads(body)
        print(body["value"])
        return True

    def consumer_callback(self, channel, method, properties, body):
        result = self.execute(body)
        if channel.is_open:
            if result:
                channel.basic_ack(delivery_tag=method.delivery_tag)   # 发送ack
            else:
                channel.basic_nack(delivery_tag=method.delivery_tag, multiple=False, requeue=True)
        if not channel.is_open:
            print("Callback 接收频道关闭，无法ack")

    def start_consumer(self):
        self.reconnect()
        while True:
            try:
                self.channel.start_consuming()  #启动消息接受 进入死循环
            except ConnectionClosed as e:
                print("ConnectionClosed: ", e)
                self.reconnect()
                time.sleep(2)
            except ChannelClosed as e:
                print("ChannelClosed: ", e)
                self.reconnect()
                time.sleep(2)
            except Exception as e:
                print("consuming: ", e)
                self.reconnect()
                time.sleep(2)
```

## 2.基本使用

```python
# publish.py
from base import RabbitPublisher

if __name__ == '__main__':
    publisher = RabbitPublisher()
    publisher.start_publish()
```

```python
# consumer.py
from base import RabbitComsumer

if __name__ == '__main__':
    consumer = RabbitComsumer()
    consumer.start_consumer()
```