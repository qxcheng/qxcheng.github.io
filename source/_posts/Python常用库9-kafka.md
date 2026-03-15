---
categories:
- - Python
date: 2024-04-17 14:07:50
description: 'Python常用库9-kafka'
id: '128'
tags:
- python
- 消息队列
title: Python常用库9-kafka
---


## 1.封装基类

```python
from kafka import KafkaProducer, KafkaConsumer
from kafka.errors import kafka_errors
import traceback
import json

class KafkaProducerSDK(object):
    def __init__(self, hostname) -> None:
        self.hostname = hostname
        self.producer = KafkaProducer(
            bootstrap_servers=[self.hostname], 
            key_serializer=lambda k: json.dumps(k).encode(),
            value_serializer=lambda v: json.dumps(v).encode()
        )

    def produce(self, message):
        future = self.producer.send(
            'mykafka',
            key='count_num',  # 同一个key值，会被送至同一个分区
            value=message,
            partition=0)      # 向分区0发送消息
        print("send {}".format(message))
        try:
            future.get(timeout=10)  # 监控是否发送成功           
        except kafka_errors:        # 发送失败抛出kafka_errors
            print(traceback.format_exc())

class KafkaConsumerSDK(object):
    def __init__(self, topic, hostname) -> None:
        self.topic = topic
        self.hostname = hostname
        self.consumer = KafkaConsumer(
            self.topic, 
            bootstrap_servers=[self.hostname],
            auto_offset_reset='latest'   # earliest
        )

    def consume(self):
        for message in self.consumer:
            if message:
                print ("%s:%d:%d: key=%s value=%s" % (message.topic, message.partition, message.offset, message.key, message.value))
            else:
                print("no message")
```

## 2.基本使用

```python
# producer.py
from base import KafkaProducerSDK

if __name__ == '__main__':
    producer = KafkaProducerSDK('192.168.0.34:9092')
    for i in range(10):
        producer.produce(i)
```

```python
# consumer.py
from base import KafkaConsumerSDK

if __name__ == '__main__':
    consumer = KafkaConsumerSDK('mykafka', '192.168.0.34:9092')
    consumer.consume()
```