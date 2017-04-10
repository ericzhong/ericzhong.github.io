---
layout: post
title: Linux 下的并发服务器
tag: 高并发
category: 计算机
---



# 多进程

`server.c`：

```c
#include <sys/types.h>
#include <sys/socket.h>  // socket
#include <netdb.h>       // sockaddr_in
#include <stdio.h>       // printf
#include <string.h>      // strlen
#include <unistd.h>      // write, close, fork, sleep
#include <signal.h>      // signal

int main(int argc, char *argv[])
{
	int sockfd = 0, connfd = 0, n = 0, pid = 0;
	socklen_t clen;
	struct sockaddr_in addr, caddr;
	char buf[INET_ADDRSTRLEN];
	char html[] = "HTTP/1.0 200 OK\nContent-type: text/html\nContent-length: 6\n\nhello\n";
	
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
	signal(SIGCHLD, SIG_IGN);       // 让系统回收子进程
	
	while (1) {
		printf("waiting...\n");
		clen = sizeof(caddr);
		connfd = accept(sockfd, (struct sockaddr*)&caddr, &clen);

		if (fork() == 0) {
			printf("Port %d\n", ntohs(caddr.sin_port));
			close(sockfd);
			write(connfd, html, strlen(html));
			close(connfd);
			return 0;
		}

		close(connfd);
	}
}
```



# 多线程

`server.c`：

```c
#include <stdio.h>         // printf
#include <string.h>        // strlen
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>
#include <unistd.h>        // write, close
#include <signal.h>        // signal
#include <pthread.h>       // pthread_*

void func(void *sock)
{
	int s = *((int *) sock);
	char html[] = "HTTP/1.0 200 OK\nContent-type: text/html\nContent-length: 6\n\nhello\n";

	pthread_detach(pthread_self());	  // 退出后自动回收资源
	write(s, html, strlen(html));
	close(s);
}

int main(int argc , char *argv[])
{
	int sockfd = 0, connfd = 0, n = 0;
	pthread_t tid = 0;
	socklen_t clen;
	struct sockaddr_in addr, caddr;
	char buf[INET_ADDRSTRLEN];
	
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
	
	while (1) {
		printf("waiting...\n");
		clen = sizeof(caddr);
		connfd = accept(sockfd, (struct sockaddr*)&caddr, &clen);
		printf("Port %d\n", ntohs(caddr.sin_port));
		pthread_create(&tid, 0, (void *)func, (void *)&connfd);
	}
}
```

