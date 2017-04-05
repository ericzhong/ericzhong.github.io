---
layout: post
title: 上海竞拍车牌的高并发场景
tags: 高并发
category: 计算机
---



上海汽车牌照竞拍的规则比较啰嗦，根据之前的成功经验，大致的策略是这样的：10:30-11:00 先按照系统指示价出一次价，这就拿到了竞拍资格。然后 11:00-11:30 实际上只有一次出价机会，就在最后 30 秒，竞拍价的均值从 11:29:30 秒就开始越涨越快，到 11:29:45 秒时，把当前均价加上 700、800 或 900 元，具体值参照上一期，然后迅速提交，因为还要输入验证码，所以当按下提交按钮的时候就已经 11:29:53 左右了，然后进度条走几秒，报价能在 11:29:58 左右进入系统就很好了，因为均价一直在涨，所以提交价在截止点的均价正负 300 区间内才算有效，所以既要保证竞拍价在有效区间，又要尽可能早的提交（相同出价者先出价的优先），这就是一个权衡了。

最近几期都是约 25w 人竞拍 1w 块牌。假设都在最后 10 秒提交，那么平均 TPS 为 2.5w/s。然后系统要不断计算均值，客户端每秒更新一次，平均 TPS 为 25w/s。当竞拍截止，尽快返回竞拍状态（成功或失败），实际等待时间在 10 分钟以内。

实验系统技术选型 lighttpd + web.py + redis + mysql。mysql 作为持久存储，截止竞拍后，计算最终的成功者并保存排序后的清单。



# lighttpd + fastcgi + web.py + redis

安装 lighttpd、web.py、redis：

```sh
yum install lighttpd
pip2 install web.py                # 只能在 Python 2 上运行，因为很多依赖库都基于 Python 2
yum install redis python-redis
```

在 `/etc/lighttpd/modules.conf` 中解除注释，使用 fastcgi：

```sh
include "conf.d/fastcgi.conf"
```

在 `/etc/lighttpd/conf.d/fastcgi.conf` 文件中增加配置：

```ruby
fastcgi.server = (
    "/" =>
    ((
        "socket" => "/tmp/fastcgi.socket",
        "bin-path" => "/var/www/lighttpd/index.py",
        "max-procs" => 10,
        "bin-environment" => ( "REAL_SCRIPT_NAME" => "" ),
        "check-local" => "disable"
    ))
)
```

配置 `/etc/redis.conf`：

```yaml
daemonize yes     # 在后台运行
save ""           # 关闭持久化
```

启动 redis 服务：

```sh
redis-server  &
```

Redis 初始化脚本 `init.py`：

```s
#!/usr/bin/env python
import redis
conn = redis.Redis()
conn.hmset('global', {'avg':87300, "tid":0})
```

测试一下：

```
python
>>> import redis
>>> conn = redis.Redis()
>>> conn.hmget('hash-key', ['avg'])
['87300']
```

`index.py`：

```python
#!/usr/bin/env python

import web
import redis
import random
import time


urls = (
    '/', 'index'
)

homepage = '''
<html>
  <form action="/" method="post">
    <p>Price: <input type="number" name="price" /></p>
    <input type="submit" value="Submit" />
  </form>
</html>
'''

result_page = '''
<html>
  <p>TID: {tid}</p>
  <p>Price: {price}</p>
  <p>Avg: {avg}</p>
  <p>Status: {status}</p>
</html>
'''

class index:
    def GET(self):
        return homepage

    def POST(self):
        i = web.input()
        price = int(i.price)

        conn = redis.Redis()
        avg, tid = [int(i) for i in conn.hmget('global', ['avg','tid'])]

        if price<(avg-300) or price>(avg+300):
            status = "refused"
            tid = None
        else:
            status = "accepted"
            conn.hmset(tid, {'price':price, 'avg':avg,
                    'time':time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(time.time()))})
            conn.hincrby('global', "tid", 1)

        return result_page.format(tid=tid, price=price, status=status, avg=avg)

if __name__ == "__main__":
    app = web.application(urls, globals())
    app.run()
```



# 后台每秒更新一次均价

`update-avg.py`：

```python
#!/usr/bin/env python

import redis
import time


if __name__ == "__main__":
    conn = redis.Redis()
    while True:
        n = 0
        tid, avg = [int(i) for i in conn.hmget('global', ['tid','avg'])]
        for i in xrange(0, tid):
            n += int(conn.hmget(i, 'price')[0])
        n /= tid
        if n != avg:
            conn.hmset('global', {'avg':n})
        # print conn.hmget('global', ['avg'])
        time.sleep(1)
```



# 压力测试

`test.sh`：

```sh
#!/usr/bin/bash

n=87300
for((i=n-500; i<n+500; i+=100))
do
    {
       post="post_${i}.txt"
       echo "price=${i}" > $post
       ab -n 2000 -c 200 -p $post -T "application/x-www-form-urlencoded" http://192.168.100.2:8000/ > ${i}.log
    } &
done
wait
```

或者执行单行语句：

```sh
# Post
ab -n 20000 -c 20000 -p post_87300.txt -T "application/x-www-form-urlencoded" http://*.*.*.*:8000/

# Get
ab -n 20000 -c 20000 http://192.168.100.2:8000/
```

`post_*.txt`：

```
price=87300
```

因为资源限制，ab 可能会报错：

```
socket: Too many open files (24)       # 默认 1024
```

把限制设置得更高一点即可：

```sh
$ ulimit -a
open files                      (-n) 1024
$ ulimit -n 500000
```



# 失败原因

如果 `ab` 输出中的 `Failed requests` 不为 0，就应该查看日志，找到原因：

* 端口不够用。端口最多 64511 (1024~65535) 个。大量端口被 `TIME_WAIT` 状态的 TCP 连接占用，默认 1 分钟才释放。
* FastCGI 进程不够。每个进程大约可以处理 1000 个并发，超载后就会报错 `overload`，然后丢掉后来的请求。
* open files 数太小。ab 的并发量超过了可以打开的 (socket) 文件数。
* CPU 负载太大。尽量让负载小于 `CPU 核心数 * 75%`。
* 内存不够。一个线程约占 0.2M 的空间。




# 性能调优

```sh
top -H -d1                    # 观察线程数和内存的变化，每秒刷新。反复输入 'E' 可以切换内存单位。
netstat -tnlp                 # 查看 tcp 监听端口占用
taskset -pc 18272             # 查看进程的 CPU 亲和力
taskset -pc 0-3 18272         # 设置进程的 CPU 亲和力

netstat -tn | grep "192.168.100.2:8000" | wc -l     # lighttpd 的连接数
netstat -tn | grep "127.0.0.1:6379" | wc -l         # redis 的连接数

# 只作用于当前终端
ulimit -n 500000                  # 最大文件数
ulimit -u 500000                  # 最大进程数

# 解决因 TIME_WAIT 耗尽端口导致的连接失败
sysctl -w net.ipv4.tcp_timestamps=1         # 默认值
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_tw_recycle=1
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
while [ 1 ]; do ss -tan state time-wait|wc -l; sleep 1; done
```

Redis 使用连接池：

```python
import redis
pool = redis.ConnectionPool(host='127.0.0.1', port=6379, db=0)
conn = redis.Redis(connection_pool=pool)
```



# 端口

端口号是 16 位的，且端口 0 代表所有端口，因此可用的只有 65535 个。

0~1023 是公认端口，需要 Root 权限才能绑定。（不能占用）

1024~49151 是注册端口，供一些专门的服务使用，不需要 Root 权限就能绑定。（有需要可以占用）

49152~65535 是动态端口。（随便使用）

公认端口和注册端口都是由 IANA 分配和管理的。



# Troubleshooting

#### ConnectionError: Error 99 connecting to localhost:6379. Cannot assign requested address.

6379 是 Redis 的监听端口。这时 lighttpd 的连接数约 2w，redis 的连接数约 3w。可能是系统清理端口的速度不够快。

如下设置可解决：

```sh
sysctl -w net.ipv4.tcp_tw_reuse=1
sysctl -w net.ipv4.tcp_tw_recycle=1
```



---



> 未完，待续 ...



