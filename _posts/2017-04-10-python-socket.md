---
layout: post
title: "Python 网络编程"
tag: linux
category: 计算机
---



# Hello World

`server.py`：

```python
#!/usr/bin/env python

import socket

s = socket.socket()
s.bind(('127.0.0.1', 8000))
s.listen(5)

while True:
    conn, addr = s.accept()
    while True:
        data = conn.recv(1024)
        if not data: break    # Broken pipe or Disconnected
        conn.sendall(data)
    conn.close()
```

`client.py`：

```python
#!/usr/bin/env python

import socket

s = socket.socket()
s.connect(('127.0.0.1', 8000))
s.send("Hello")
data = s.recv(1024)
print data
s.close()
```



# HTTP 响应

`server.py`：

```python
#!/usr/bin/env python

import socket

# 响应头和数据之间要有一个空行。数据最后有一个回车 (6 bytes)
html = '''\
HTTP/1.1 200 OK
Content-type: text/html
Content-length: 6

hello
'''

s = socket.socket()
s.bind(('127.0.0.1', 8000))
s.listen(5)

while True:
    conn, addr = s.accept()
    data = conn.recv(1024)
    if data:
        conn.sendall(html)
    conn.close()
```

可以用 curl 测试：

```sh
curl http://127.0.0.1:8000/
```



# select

`server.py`：

```python
#!/usr/bin/env python

import socket
import select
import time
import sys

html = '''\
HTTP/1.1 200 OK
Content-type: text/html
Content-length: 6

hello
'''

s = socket.socket()
s.setblocking(0)
s.bind(('127.0.0.1', 8000))
s.listen(5)

inputs = [s]
outputs = []
excepts = [s]

def remove_conn(conn, lists):
    for l in lists:
        if conn in l:
            l.remove(conn)

try:
    while inputs:
        readable, writable, exceptional = select.select(inputs, outputs, excepts)
        all_lists = [readable, writable, exceptional, inputs, outputs, excepts]
        
        for i in readable:
            if i is s:
                conn, addr = i.accept()
                conn.setblocking(0)
                inputs.append(conn)
            else:
                data = i.recv(1024)
                if not data:            # Broken pipe or Disconnected
                    remove_conn(i, all_lists)
                    i.close()
                    continue
                if i not in outputs:
                    outputs.append(i)
        
        for i in writable:
            i.send(html)
            remove_conn(i, all_lists)
            i.close()
      
        for i in exceptional:
            if i is s:
                sys.exit(1)
            remove_conn(i, all_lists)
            i.close()

finally:
    s.close()
```



# epoll

只能用于 Linux。

Edge Trigger：边沿触发。状态发生改变才通知。  
Level Trigger：水平触发。状态满足就通知（默认选项）。

`server.py`：

```python
#!/usr/bin/env python

import socket, select

html = '''\
HTTP/1.1 200 OK
Content-type: text/html
Content-length: 6

hello
'''

s = socket.socket()
s.bind(('0.0.0.0', 8080))
s.listen(1)
s.setblocking(0)

epoll = select.epoll()
epoll.register(s.fileno(), select.EPOLLIN)

try:
    conns = {}
 
    while True:
        events = epoll.poll(1)
 
        for fd, event in events:
            if fd == s.fileno():
                conn, addr = s.accept()
                conn.setblocking(0)
                epoll.register(conn.fileno(), select.EPOLLIN)
                conns[conn.fileno()] = conn
 
            elif event & select.EPOLLIN:
                data = conns[fd].recv(1024)
                epoll.modify(fd, select.EPOLLOUT)
 
            elif event & select.EPOLLOUT:
                conns[fd].send(html)
                epoll.unregister(fd)
                conns[fd].close()
                del conns[fd]
 
            elif event & select.EPOLLHUP:
                epoll.unregister(fd)
                conns[fd].close()
                del conns[fd]
 
finally:
    epoll.unregister(s.fileno())
    epoll.close()
    s.close()
```



# socket 配置

```python
# IPPROTO_TCP: TCP 层选项
# SOL_SOCKET: socket 层选项
s.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
s.setblocking(0)
```









