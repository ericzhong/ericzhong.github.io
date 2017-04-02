---
layout: post
title: "SysVinit、Upstart、Systemd"
tag: linux
category: 计算机
---



#### SysVinit

​内核初始化后的第一步便是启动 `PID=1` 的 `init` 进程。它是所有进程的祖先，负责创建其它进程。大多数 Linux 发行版的 init 程序是和 `System V` 相兼容的，被称为 `SysVinit`。

​它靠 runlevel 来标识运行模式，`/etc/inittab` 中的 `initdefault` 值表示默认的 `runlevel`。总共有 7 个运行级别，0-关机、1-单用户、3-终端、5-图形、6-重启。

​然后执行 `/etc/rc.d/rc.sysinit`，它会激活 udev 和 selinux、加载时钟、设置主机名、检查并挂载所有文件系统、清除过期的 locks 和 PID 文件等等。

然后根据运行级别执行 `/etc/rc.d/rcN.d/` 目录下的脚本，脚本名类似 S10network，字母 S 表示启动期间执行的脚本，数字是执行顺序，从最小的开始执行。最后执行的是 `S99local`，这个脚本供用户定制用，它是个软链接，指向 `rc.local`。

​最后，关闭系统时 `sysinit` 还要负责清理工作。它会执行 `/etc/rc.d/rcN.d/` 目录下所有以字母 K 开头的脚本。它们负责安全的停止服务，卸载所有文件系统等等。

SysVinit 有一些命令，`halt、init、last、mesg、pidof、reboot、runlevel、shutdown、wall` 等。RHEL 在 SysVinit 的基础上开发了 `initscripts` 包，包含一系列启动脚本，还提供了我们常用的 `service`、`chkconfig` 命令。

SysVinit 的脚本必须按照顺序逐个执行，而且必须一次将可能需要的服务全部启动，这在服务器上是没问题的，可是在移动设备上就不方便了，因为很多外设是热插拔的，也就无法预知用户将使用什么样的服务。而且移动设备要求更快的启动速度，逐个执行脚本的同步方式太慢了。

所以，后来的 Upstart、Systemd 都是为了解决如下问题：

- 更快的启动速度：开机时启动尽量少的服务、用二进制程序替代脚本、用异步方式并行启动尽量多的任务。
- 支持设备热插拔：根据需要随时启动服务。




#### Upstart

Upstart 是基于任务的，且任务通过事件来触发，即异步模式，这样就可以解决热插拔识别问题，等到需要时才启动相应的服务，而且异步模式提升了启动的速度。

Upstart 兼容 SysVinit。它是 Canonical 公司开发的，最开始用于 Ubuntu 系统，后来又被 Systemd 取代了。

Upstart 提供的命令是：

```sh
$ initctl list|start|stop|restart|reload
$ runlevel
$ start|stop|restart|reload job
```



#### Systemd

Systemd 的并发度更高，因此启动速度比 Upstart 更快。它用 `target` 替代了 runlevel，target 之间可以继承。

另外，Upstart 主要靠 `strace` 来跟踪子进程，而 Systemd 使用 `CGroup`，更快速。

Systemd 提供的命令是：

```sh
$ systemctl start|stop|restart|reload|status|enable|disable|... target
$ systemctl reboot|poweroff|suspend|hibernate|hybrid-sleep
```



#### 参考

- [https://refspecs.linuxbase.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/runlevels.html](https://refspecs.linuxbase.org/LSB_5.0.0/LSB-Core-generic/LSB-Core-generic/runlevels.html)
- [https://en.wikipedia.org/wiki/Systemd](https://en.wikipedia.org/wiki/Systemd)
- [https://en.wikipedia.org/wiki/Upstart](https://en.wikipedia.org/wiki/Upstart)
- [浅析 Linux 初始化 init 系统，第 1 部分: sysvinit](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init1/index.html)
- [浅析 Linux 初始化 init 系统，第 2 部分: UpStart](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init2/index.html)
- [浅析 Linux 初始化 init 系统，第 3 部分: Systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/index.html)
