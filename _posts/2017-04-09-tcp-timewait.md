---
layout: post
title: "TIME_WAIT 与 CLOSE_WAIT"
tag: 网络
category: 计算机
---



# 什么是 TIME_WAIT、CLOSE_WAIT？

本地压测时一般会出现大量 `TIME_WAIT`，可以用如下命令查看：

```sh
netstat -na | grep 8000             # 服务端口可能不同
ss -tan state time-wait
```

先看看 TCP 连接的打开、关闭原理：

![tcp-open-close](http://omy6w6iwk.bkt.clouddn.com/tcp-open-close.png)

主动关闭方（即先调用 `close()` 的那一方）在收到对方（调用 `close()` 后产生）的  `FIN` 后，会处于 `TIME_WAIT` 状态。而被动关闭方在收到第一个 FIN 后就会处于 `CLOSE_WAIT` 状态。

简单的说，**处于 `TIME_WAIT` 状态的肯定是主动关闭方，处于 `CLOSE_WAIT` 状态的肯定是被动关闭方**。

MSL (Maximum segment lifetime) 即 TCP 分组在网络上的存活时间，TIME_WAIT 状态在 CentOS 7 上会持续 1 分钟，且它的值是 2MSL，那么 MSL=30s。

为什么要 TIME_WAIT，且为什么一定是 2MSL 呢？

1. 怕最后一个 ACK 对方没收到。

   因为网络延迟或者其它原因，最后一个 ACK 可能对方没及时收到，超时后，对方会重发一个 FIN，这时可以再回一个 ACK。

2. 等老的分组在网络上消失。

   假设没有 TIME_WAIT 状态，连接在断开后可以马上重用，就可能出现如下问题：一个连接中的第 n 个分组在网络上延迟了，然后断开，再连接，在等待第 n 个分组时，它刚好到了，这时接收的其实是上一个连接的第 n 个分组，就产生了错误。要避免，只需要等待一会儿，让这个分组自动过期，而 2MSL 正好是一去一回的最长时间。

至于 `CLOSE_WAIT` 一般是自己的程序有 bug。在收到对方的 `FIN` 后，再调用 `read()` 就会返回 0，这时应该调用 `close()` 来关闭连接，但如果没有调用，就会一直处于该状态。



# 为什么压测会产生 TIME_WAIT？

如果在本地做实验会发现，用浏览器或 curl 访问局域网的服务一般不会产生 `TIME_WAIT`，但是压测时就一定会有。

符合规范的客户端在读完数据后都会主动关闭连接，只有主动关闭方才会处于 TIME_WAIT 状态，所以在局域网的客户端系统上是可以查看到 TIME_WAIT 的，但是服务器端则没有。而数据的长度是通过 HTTP 协议中的 `Content-length:` 响应头告知客户端的。

以 curl 为例，如果服务器端响应的 Content-length 值比实际数据小，那么 curl 读取 Content-length 长度的数据后就会主动关闭连接。但如果比实际数据大或者根本就没收到该响应头，则会一直处于读等待状态，这时就必须服务器端来主动关闭连接。

但是在压测时，客户端估计是反应太慢，服务器等待一段时间后就主动关闭了，所以才有会有大量的 `TIME_WAIT`，这个猜测可通过如下的并发服务器代码来验证：

```c
#include <stdio.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <signal.h>

int main(int argc , char *argv[])
{
	int sockfd = 0, connfd = 0, n = 0, pid = 0;
	socklen_t clen;
	struct sockaddr_in addr, caddr;
	char buf[INET_ADDRSTRLEN], rbuf[1025];
	char html[] = "HTTP/1.1 200 OK\nConnection: close\nContent-type: text/html\nContent-length: 6\n\nhello\n";
	
	sockfd = socket(AF_INET , SOCK_STREAM , 0);
	if (sockfd == -1) {
		perror("Error: socket()");
        return 1;
	}
	
	addr.sin_family = AF_INET;
	addr.sin_addr.s_addr = INADDR_ANY;
	addr.sin_port = htons(8000);
	
	if (bind(sockfd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
		perror("Error: bind()");
		return 1;
	}
	
	listen(sockfd, 5);
	signal(SIGCHLD, SIG_IGN);
	
	while (1) {
        printf("waiting...\n");
        clen = sizeof(caddr);
	    connfd = accept(sockfd, (struct sockaddr*)&caddr, &clen);

		if (fork() == 0) {
			printf("Port %d\n", ntohs(caddr.sin_port));
			close(sockfd);
			read(connfd, &rbuf, sizeof(rbuf)+1);
			write(connfd, html, strlen(html));
			// usleep(1000000);
			close(connfd);
			return 0;
		}

	close(connfd);
    }
}
```

用客户端访问这个并发服务器时，如果将 `usleep()` 这一行注释掉，那么 Server 端一定会产生 `TIME_WAIT`，因为它在写完后就立即主动关闭连接了。通过不断调整 `usleep()` 的参数，就可以知道等待客户端的 FIN 需要多久，测试值如下：

* curl：`for((i=0;i<100;i++)); do curl URL; done` 至少需要等待 400 微秒。
* Siege：`siege -r 100 -c 1 URL` 至少需要 400 微秒，而 `siege -r 10 -c 10` 则至少需要 7000 微秒。
* Chrome：40000 微妙（走互联网）。
* ab：不会主动关闭连接。

curl 和 Siege 在读完数据后都会主动关闭连接，服务器只要等待约 400 微秒就可以收到 FIN。但是当并发请求时，Siege 的反应就慢了很多。

curl 和 Siege 单并发都不会让 Lighttpd 产生 TIME_WAIT，但是 Siege 的多并发就一定会，所以 Lighttpd 的等待时间应该在 400~7000 微秒之间，当然，这也有可能只是短连接的值。而对 `ab` 的测试结果是，它不管怎样都不会主动关闭连接，`Content-length:` 响应头是否存在也没影响。

所以，总结一下：**如果想避免服务器端产生大量的 `TIME_WAIT`，就必须让客户端来关闭连接，要么服务器端多等一会儿，要么客户端能快一点调用 `close()`**。



# 复用 TIME_WAIT 连接

socket 连接是一个五元组，即 `协议、源 IP、源端口、目的 IP、目的端口`。对于 HTTP 服务来说，使用的是 TCP 协议，因此也有叫四元组的。

服务器端处于 TIME_WAIT 状态的连接，就是一个五元组信息，在 2MSL 时间内该连接是不能重用的，也就说不能有重复的五元组信息出现。比如，服务器端产生了 TIME_WAIT 连接，这时关闭服务，再启动就会报错 “端口已被占用”，只有等到 TIME_WAIT 连接关闭后才能在相同的端口上再次启动服务。

生产环境下，请求走的是互联网，按照上面的测试结果，一定会产生 TIME_WAIT，即服务器肯定等不及客户端来关闭连接就已经先关闭了。但是客户端每次连接都会用不同的端口，即不会产生重复的五元组，所以正常访问一般不会产生问题。

维护五元组信息对服务器端来说只是占用一点空间而已，可忽略不计。

```sh
$ dmesg | grep "TCP"
[    0.254985] TCP established hash table entries: 131072 (order: 8, 1048576 bytes)
[    0.255221] TCP bind hash table entries: 65536 (order: 8, 1048576 bytes)
[    0.255352] TCP: Hash tables configured (established 131072 bind 65536)
```

但是，用单台机器做压测的话，可用的端口就那么多，所以每分钟内可以建立的并发连接数就很有限了（因为五元组中只有源端口改变了）。所以减少 TIME_WAIT 有时候也是必要的。

重用 TIME_WAIT 连接的设置如下：

```sh
$ sysctl -w net.ipv4.tcp_timestamps=1    # 默认就是打开的
$ sysctl -w net.ipv4.tcp_tw_reuse=1
$ sysctl -w net.ipv4.tcp_tw_recycle=1
```

`reuse` 表示是否重用 TIME_WAIT 连接，即内核中保存五元组信息的数据结构是否可重用。

`recycle` 表示是否开启 TIME_WAIT 的快速回收，这个回收机制是建立在 `net.ipv4.tcp_timestamps` 选项上的，它默认开启，因此系统会记录最近一次收、发分组中的时间戳（总共两个值），然后按照 RTO 时间来回收，比 2MSL 小多了，所以也是一种权衡吧。

> TCP 连接的两端都会维护一个 RTT (Round-TripTime) 值，就是一个分组一来一回的平均时间，是动态调整的。而 RTO (Retransmission TimeOut) 值就是根据 RTT 计算出来的超时重传时间，因此也是动态的。

有了最近的收、发分组的时间戳，再加上 RTO 值，自然就知道什么时候可以回收，因此 `net.ipv4.tcp_timestamps` 选项必须开启，才能使 TIME_WAIT 快速回收机制正常工作。

如果老的分组真的晚到了呢？如果某分组的时间戳小于最新的时间戳将被丢弃。

另外，**快速回收是工作在主动关闭连接的这一方，也只有这一方才有 TIME_WAIT 状态的连接需要回收**。

**快速回收不能用于 NAT 环境**，因为快速回收机制会将小于最新时间戳的分组丢弃，而 NAT 只是简单转发，内网的多个主机通过同一个 IP 地址出去，源包的发送时间戳不可能单调递增，所以会导致大量的包被丢弃。

```sh
netstat -s | grep timestamp        # 查看因时间戳过小而丢弃的包
```

系统中还有一个选项 `net.ipv4.tcp_max_tw_buckets` 用来设置 TIME_WAIT 连接数的上限，多余的会自动清除。