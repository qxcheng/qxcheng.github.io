---
categories:
- - Python
date: 2024-04-17 11:17:04
description: 'Python进阶7-网络编程'
id: '108'
tags:
- python
title: Python进阶7-网络编程
---


## 1.TCP

```Python
#1 服务端 server.py
import socket    

host = '127.0.0.1'  # 设置ip
port = 9000         # 设置端口

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # 创建socket对象
s.bind((host, port))     # 绑定ip和端口
s.listen(5)              # 等待客户端连接
print("开始监听...")

while True:
    c, addr = s.accept() # 建立客户端连接
    print('客户端连接地址：', addr)

    data = c.recv(2048)
    print("接收到消息：", data.decode('utf-8'))

    c.send(b'Welcome to connect!')
    c.close()            # 关闭连接

#2 客户端 client.py
import socket  

host = '127.0.0.1' # 设置ip
port = 9000        # 设置端口

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # 创建socket对象
s.connect((host, port))

s.send(b'Hello')

data, addr = s.recvfrom(1024)
print(data.decode('utf-8'))

s.close() 
```

## 2.UDP

```Python
#1 server.py
import socket

host = '127.0.0.1' # 设置ip
port = 9000        # 设置端口

s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)  # udp协议
s.bind((host, port))

while True:
    data, addr = s.recvfrom(1024)
    print('server收到来自 {} 的消息：'.format(addr), data)
    s.sendto(data.upper(), addr)

s.close()

#2 client.py
import socket

host = '127.0.0.1' # 设置ip
port = 9000        # 设置端口

c = socket.socket(socket.AF_INET,socket.SOCK_DGRAM)

c.sendto(b'hello', (host, port))
data, addr = c.recvfrom(1024)
print('客户端收到来自 {} 的消息：'.format(addr), data)

c.close()
```

## 3.websocket

```Python
pip install websocket_client
pip install websocket-server

#1 server
from websocket_server import WebsocketServer

# 当新的客户端连接时会提示
def new_client(client, server):
    print("新的客户端连接:%s" % client['id'])
    server.send_message_to_all("Hey all, a new client has joined us")

# 当旧的客户端离开
def client_left(client, server):
    print("客户端%s断开" % client['id'])

# 接收客户端的信息
def message_received(client, server, message):
    print("Client(%d) said: %s" % (client['id'], message))
    server.send_message_to_all(message)

if __name__ == '__main__':
    server = WebsocketServer(8000, host='127.0.0.1')
    server.set_fn_new_client(new_client)
    server.set_fn_client_left(client_left)
    server.set_fn_message_received(message_received)
    server.run_forever()

#2 client
import websocket
import threading, time
from websocket import create_connection

ws = create_connection("ws://127.0.0.1:8000/")

def recv():
    start = time.time()
    try:
        while ws.connected:
            result = str(ws.recv())
            print("Recv:", result)
    except websocket.WebSocketConnectionClosedException:
        print("receive result end")
        print("耗时：%s" % (time.time() - start))

if __name__ == '__main__':
    t = threading.Thread(target=recv)
    t.start()

    for i in range(5):
        print("Send: {}".format(i))
        ws.send("Hello, {}!".format(i))
        time.sleep(1)

    ws.close()
```