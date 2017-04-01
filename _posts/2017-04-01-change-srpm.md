---
layout: post
title: 修改 SRPM 包
tag: linux
category: 计算机
---



> 环境：CentOS 7
>
> 样例包：tree



#### 原理

红帽系的包有两种，RPM 是二进制安装包，SRPM 是源码包。这点跟 Debian 系的不太一样，Debian 系只有二进制的 DEB 安装包，源码则是多个文件，而非一个整包。

SRPM 包也可以安装，会安装到如下目录框架下，这个框架是自动生成的，是红帽用来改包、打包的标准结构，不是 SRPM 包本身的目录。

```sh
~/rpmbuild/
├── BUILD/            # 解压并打上 Patch 后的源码
├── BUILDROOT/
├── RPMS/             # 编译出的 RPM 包
├── SOURCES/          # 从 SRPM 包中解出的 tar 包、Patch 文件等等
├── SPECS/            # 包描述文件
└── SRPMS/            # 编译出的 SRPM 包
```

修改 SRPM 包的基本步骤：

1. 安装 SRPM 包 (`rpm -ivh`)。其实就是把它解开，然后将描述文件 `*.spec` 放到 `SPECS/` 目录下，剩下的 tar 包、Patch 文件等全部放到 `SOURCES/` 目录下。
2. 把源码 tar 包再解开并打上所有 Patch (`rpmbuild -bp`)。最终产出都在 `BUILD/` 目录下。
3. 修改源码并生成新的 Patch，再放到 `SOURCES/` 目录下。
4. 将新的 Patch 添加到描述文件 `*.spec` 中。
5. 在 `*.spec` 中添加 changelog。
6. 编译出 RPM包、SRPM 包。



#### 样例

首先，安装一些编译、修改包会用到的工具，以及 tree 包的编译依赖：

```sh
$ sudo yum install rpm-build rpmdevtools      # 安装需要的工具
$ sudo yum-builddep tree                      # 安装编译依赖
```

注意：编译依赖是从仓库获取的，也就是老版本的包的依赖，如果新增了依赖，就得手动安装，并且一定要添加到 spec 文件中。



然后下载源码包，解压、打上 Patch，最后复制一份源码方便后面生成 Patch：

```sh
$ yumdownloader --source tree
tree-1.6.0-10.el7.src.rpm                                  |  56 kB   00:01     
$ rpm -ivh tree-1.6.0-10.el7.src.rpm
$ cd ~/rpmbuild/SPECS/
$ rpmbuild -bp tree.spec                      # 解开源码包并打上 Patch
$ cd ~/rpmbuild/BUILD                         # 源码解压在这个目录下
$ cp -af tree-1.6.0/ tree-1.6.0.orig          # 备一份，后面生成 Patch 用
```



可以开始修改源码了，这里为了测试，把 tree 包的 README 文件覆盖掉，最后生成 Patch 文件：

```sh
$ cd tree-1.6.0
$ echo "just for test" > README
$ cd ~/rpmbuild/BUILD
$ diff -Naur tree-1.6.0.orig/ tree-1.6.0 > ~/rpmbuild/SOURCES/tree-change-readme.patch
```



接下来，将新增的 Patch 加入到描述文件 `~/rpmbuild/SPECS/tree.spec` 中，照葫芦画瓢就行：

```sh
# 这里是告诉它多了一个 Patch 文件
Patch5: tree-fixbufsiz.patch
Patch6: tree-change-readme.patch          # 新增行

# 这里是告诉它怎么打这个 Patch
%patch5 -p1 -b .fixbufsiz
%patch6 -p1 -b .change-readme             # 新增行
```



可以添加 changelog 了，用下面的命令可以插入 changelog 到 `tree.spec` 中，并自动提升修订号：

```sh
$ rpmdev-bumpspec -c "change readme" -u "Eric Zhong <ericiszhongwenjia@qq.com>" -r ~/rpmbuild/SPECS/tree.spec
```



检查一下 `tree.spec` 中的 changelog 部分。注意修订号的变化 `10 -> 10.1`，如果想 `10 -> 11` 就去掉上面的 `-r` 选项。

    %changelog
    * Sat Apr 01 2017 Eric Zhong <ericiszhongwenjia@qq.com> - 1.6.0-10.1
    - change readme
    
    * Fri Jan 24 2014 Daniel Mach <dmach@redhat.com> - 1.6.0-10
    - Mass rebuild 2014-01-24


编译出 RPM 包，如果还想同时编译出 SRPM 包就换成 `-ba` 选项：

```sh
$ rpmbuild  -bb tree.spec
...
Wrote: /home/tmp/rpmbuild/RPMS/x86_64/tree-1.6.0-10.el7.centos.1.x86_64.rpm
Wrote: /home/tmp/rpmbuild/RPMS/x86_64/tree-debuginfo-1.6.0-10.el7.centos.1.x86_64.rpm
...
```

注意：编译之后， `~/rpmbuild/BUILD/` 目录下的源码目录中会有很多编译产出，之后就不能再像上面例子中那样对整个目录做 diff 了。



最后，安装并验证一下修改是否成功：

```sh
$ sudo yum localinstall ~/rpmbuild/RPMS/x86_64/tree-1.6.0-10.el7.centos.1.x86_64.rpm
$ rpm -ql tree
...
/usr/share/doc/tree-1.6.0/README
...

$ cat /usr/share/doc/tree-1.6.0/README
just for test
```

