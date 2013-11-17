---
layout: post
title: OpenStack源码安装
tags: openstack cloud
category: it
---

# 安装

## 准备实验环境

安装`Ubuntu 12.04 LTS`作为实验环境

准备两台机器（All-In-One 一台也行），各两个网卡（如果使用虚拟机做实验，`eth0`全部`桥接`，用来访问Internet，`eth1`全部`Host-only`，内部网络）

`/etc/network/interfaces`配置参考如下：

	auto lo
	iface lo inet loopback
	
	auto eth0
	iface eth0 inet dhcp
	
	auto eth1
	iface eth1 inet static
	address 10.0.0.10
	netmask 255.255.255.0

为使手动配置生效，删除图形网络管理工具`NetworkManager`（也可以用图形界面配置，这里使用手动只是为了方便演示）

	dpkg -r network-manager network-manager-gnome

重启服务后设置生效

	/etc/init.d/networking restart

两台机器IP（仅供参考）：

	# Controller Node
	192.168.1.4		# external (Internet)
	10.0.0.10		# internal
	
	# Compute Node
	192.168.1.6		# external (Internet)
	10.0.0.11		# internal

两台机器都配置好`hostname`，`/etc/hosts`增加：

	10.0.0.10       controller
	10.0.0.11       compute1

##安装基本包

后面会用到的工具，以及安装过程中会需要的一些额外依赖，一次性全装了

	apt-get install vim git screen python-pip python-dev libxml2-dev libxslt-dev sheepdog

##安装NTP

为了保持时间同步，需要在所有运行OpenStack服务的Node上安装NTP

	apt-get install ntp

除`controller`外，其它所有机器新增`/etc/cron.daily/ntpdate`，内容如下：

	ntpdate controller
	hwclock -w

修改权限

	chmod a+x /etc/cron.daily/ntpdate


## 安装数据库

> Compute Node不用装`mysql-server`和作以下配置

	apt-get install mysql-client python-mysqldb mysql-server

终端弹出界面，提示输入数据库的root用户密码（例如：`111111`）

修改`/etc/mysql/my.cnf`，允许从外部访问

	#bind-address           = 127.0.0.1
	bind-address            = 10.0.0.10

删除匿名用户（第一个填`N`，不改Root密码，其它都填`Y`）

	mysql_secure_installation

重启服务

	/etc/init.d/mysql restart


## 安装消息队列

	apt-get install rabbitmq-server
	rabbitmqctl change_password guest 321321
	

## 安装Keystone

安装

	# commit 6751c7dc3b1b364e8f59d452f5702e89d4d56bcb
	git clone git://github.com/openstack/keystone.git

	cd keystone
	pip install -r requirements.txt
	python setup.py install
	cd ..

创建数据库(`keystone`)和用户(`keystone`,`keystone_password`)

	mysql -u root -p
	create database keystone;
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone_password';
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone_password';
	quit

复制配置文件

	mkdir -p /etc/keystone
	cp etc/* /etc/keystone/
	
	cd /etc/keystone
	cp keystone.conf.sample keystone.conf
	
修改`/etc/keystone/keystone.conf` (使用`openssl rand -hex 10`生成`token`随机串`fa9a647b8de836869722`)

	[DEFAULT]
	admin_token = fa9a647b8de836869722
	public_port = 5000
	admin_port = 35357
	public_endpoint = http://controller:%(public_port)s/v2.0
	admin_endpoint = http://controller:%(admin_port)s/v2.0

	log_file = keystone.log
	log_dir = /var/log/keystone

	[sql]
	connection = mysql://keystone:keystone_password@controller/keystone

	[catalog]
	driver = keystone.catalog.backends.sql.Catalog

创建日志目录

	mkdir -pv /var/log/keystone

初始化数据库

	keystone-manage db_sync
	
初始化证书`/etc/keystone/ssl/*` （参数意义：`chmod root:root`）

	keystone-manage pki_setup --keystone-user=root --keystone-group=root
	
启动keystone服务(加上`-d`参数可查看debug信息)

	keystone-all
	
设置环境变量（目前还没有账户，只能先使用`admin_token`方式验证）

	export SERVICE_TOKEN=fa9a647b8de836869722
	export SERVICE_ENDPOINT=http://controller:35357/v2.0
	
创建类型为`identity`的`service`和相应的`endpoint`

	$ keystone service-create --name=keystone --type=identity \
	--description="Keystone Identity Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |    Keystone Identity Service     |
	|      id     | 714af530900840ef88106765f13f1921 |
	|     name    |             keystone             |
	|     type    |             identity             |
	+-------------+----------------------------------+
	
	$ keystone endpoint-create --service_id 714af530900840ef88106765f13f1921 \
	--publicurl 'http://controller:5000/v2.0' \
	--adminurl 'http://controller:35357/v2.0' \
	--internalurl 'http://controller:5000/v2.0'
	
	$ keystone endpoint-list
	+----------------------------------+-----------+----------------------------+----------------------------+-----------------------------+----------------------------------+
	|                id                |   region  |         publicurl          |        internalurl         |           adminurl          |            service_id            |
	+----------------------------------+-----------+----------------------------+----------------------------+-----------------------------+----------------------------------+
	| d28d840d4be74a778578953a1a13324f | regionOne | http://controller:5000/v2.0 | http://controller:5000/v2.0 | http://controller:35357/v2.0 | 714af530900840ef88106765f13f1921 |
	+----------------------------------+-----------+----------------------------+----------------------------+-----------------------------+----------------------------------+
	
### 创建管理员（admin）账户

创建用户、角色、租户

	keystone user-create --name admin --pass admin_password
	keystone role-create --name admin
	keystone tenant-create --name admin
	keystone user-role-add --user=admin --role=admin --tenant=admin

预先为所有服务创建一个租户

	keystone tenant-create --name=service
	
为admin用户创建快速设置环境变量的脚本
	
	cat > ~/keystonerc_admin << EOF
	export OS_USERNAME=admin
	export OS_PASSWORD=admin_password
	export OS_TENANT_NAME=admin
	export OS_AUTH_URL=http://controller:35357/v2.0/
	export PS1="[\u@\h \W(keystone_admin)]\$ "
	EOF

不再使用`admin_token`的验证方式，清除环境变量

	unset SERVICE_TOKEN
	unset SERVICE_ENDPOINT
	
用admin用户测试一下 (如果keystone用`-d`参数启动，还能看到debug信息)

	source ~/keystonerc_admin

	$ keystone user-list
	+----------------------------------+-------+---------+-------+
	|                id                |  name | enabled | email |
	+----------------------------------+-------+---------+-------+
	| 94d416d8ebf34d3e97e345afcc5a2283 | admin |   True  |       |
	+----------------------------------+-------+---------+-------+

### 创建用户账户 (供参考)

创建用户、角色、租户

	keystone user-create --name USER --pass PASS
	keystone role-create --name ROLE
	keystone tenant-create --name TENANT
	keystone user-role-add --user=USER --role=ROLE --tenant=TENANT
	
为用户创建环境变量快速设置脚本

	cat > ~/keystonerc_USER << EOF
	export OS_USERNAME=USER
	export OS_PASSWORD=PASS
	export OS_TENANT_NAME=TENANT
	export OS_AUTH_URL=http://controller:5000/v2.0/
	export PS1="[\u@\h \W(keystone_USER)]\$ "
	EOF
	
测试一下（`user-list`只有管理员才有权限调用，应该会报错，但是`token-get`可以返回信息）

	source ~/keystonerc_USER
	
	$ keystone user-list
	2013-09-08 14:10:14.312 10936 WARNING keystone.common.wsgi [-] You are not authorized to perform the requested action, admin_required.
	You are not authorized to perform the requested action, admin_required. (HTTP 403)
	
	keystone token-get


## 安装Glance

安装

	# commit 153e33a8f5ae097aa1d9f0bdbf0010fc387dc22c
	git clone git://github.com/openstack/glance.git

	cd glance
	pip install -r requirements.txt
	python setup.py install
	cd ..

	# commit 518cb2508d6557f1e8f1c8c480720e46fef4bae9
	git clone git://github.com/openstack/python-glanceclient.git 

	cd python-glanceclient
	pip install -r requirements.txt
	python setup.py install
	cd ..
	
创建数据库（`glance`）和用户（`glance`,`glance_password`）

	mysql -u root -p
	create database glance;	GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance_password';	GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance_password';
	quit
	
复制配置文件

	mkdir -p /etc/glance
	cp glance/etc/* /etc/glance/

修改`/etc/glance/`下的`glance-api.conf`和`glance-registry.conf`
	
	[DEFAULT]
	sql_connection = mysql://glance:glance_password@controller/glance
	
	[keystone_authtoken]
	auth_host = controller
	auth_port = 35357
	auth_protocol = http
	admin_tenant_name = service
	admin_user = glance
	admin_password = glance_password

创建日志目录

	mkdir -p /var/log/glance

初始化数据库

	glance-manage db_sync

> 如果数据库配置不正确，可能会在当前目录生成默认数据库文件`glance.sqlite`，最好保险的方式是确认数据库`glance`中是否创建了表
>
>		mysql -u root -p
>		use glance;
>		show tables;

创建用户

	keystone user-create --name=glance --pass=glance_password
	keystone user-role-add --user=glance --tenant=service --role=admin

创建Service和Endpoint

	$ keystone service-create --name=glance --type=image --description="Glance Image Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |       Glance Image Service       |
	|      id     | c9f6e93cfd384a27bdac595be296ad4a |
	|     name    |              glance              |
	|     type    |              image               |
	+-------------+----------------------------------+	
	$ keystone endpoint-create --service_id c9f6e93cfd384a27bdac595be296ad4a \
	--publicurl http://controller:9292 \
	--adminurl http://controller:9292 \
	--internalurl http://controller:9292
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	|   adminurl  |     http://controller:9292       |
	|      id     | c052904b25ac44c785c9ecb4cd53507e |
	| internalurl |     http://controller:9292       |
	|  publicurl  |     http://controller:9292       |
	|    region   |            regionOne             |
	|  service_id | c9f6e93cfd384a27bdac595be296ad4a |
	+-------------+----------------------------------+

启动服务

	glance-api --config-file /etc/glance/glance-api.conf
	glance-registry --config-file /etc/glance/glance-registry.conf

测试一下

	source keystonerc_admin

	glance image-list
	+----+------+-------------+------------------+------+--------+
	| ID | Name | Disk Format | Container Format | Size | Status |
	+----+------+-------------+------------------+------+--------+
	+----+------+-------------+------------------+------+--------+

增加image

	wget https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img
	
	glance image-create --name="Cirros 0.3.0" --disk-format=qcow2 \
	--container-format=bare --is-public=true < cirros-0.3.0-x86_64-disk.img

查看一下

	$ glance image-list
	+--------------------------------------+--------------+-------------+------------------+---------+--------+
	| ID                                   | Name         | Disk Format | Container Format | Size    | Status |
	+--------------------------------------+--------------+-------------+------------------+---------+--------+
	| 4d1a5832-f229-4697-a889-39311f2611ef | Cirros 0.3.0 | qcow2       | bare             | 9761280 | active |
	+--------------------------------------+--------------+-------------+------------------+---------+--------+

	$ ls /var/lib/glance/images/
	4d1a5832-f229-4697-a889-39311f2611ef


##安装Nova

###All Nodes

安装

	# commit 5ac0475845e1c7ee8cc19b38d37a0af7a3f4c1fa
	git clone git://github.com/openstack/nova.git
	
	cd nova
	pip install -r requirements.txt
	python setup.py  install
	cd ..

	# commit 3978172012a601e3a623066fd724f5a6dd9e1caf
	git clone https://github.com/openstack/python-novaclient.git
	
	cd python-novaclient
	pip install -r requirements.txt
	python setup.py  install
	cd ..

安装配置文件

	cp -af  nova/etc/nova /etc
	cp /etc/nova/nova.conf.sample /etc/nova/nova.conf

###Controller Node

创建数据库(`nova`)和用户（`nova`,`nova_password`）

	mysql -u root -p
	create database nova;
	GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova_password';
	GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova_password';
	quit

修改`/etc/nova/nova.conf`

	[DEFAULT]

	sql_connection = mysql://nova:nova_password@controller/nova

	my_ip=10.0.0.10
	vncserver_listen=0.0.0.0
	vncserver_proxyclient_address=10.0.0.10

	auth_strategy=keystone
	lock_path=/var/lock/nova

	# RabbitMQ
	rpc_backend = nova.rpc.impl_kombu
	rabbit_host = controller
	rabbit_port=5672
	rabbit_userid=guest
	rabbit_password=321321

修改`/etc/nova/api-paste.ini`

	[filter:authtoken]
	auth_host = controller
	admin_tenant_name = service
	admin_user = nova
	admin_password = nova_password

创建目录

	mkdir -p /var/log/nova

初始化数据库

	nova-manage db sync

创建用户

	keystone user-create --name=nova --pass=nova_password
	keystone user-role-add --user=nova --tenant=service --role=admin

创建Service和Endpoint

	$ keystone service-create --name=nova --type=compute --description="Nova Compute Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |       Nova Compute Service       |
	|      id     | 96b9ce17137744e58094009ac23f1d63 |
	|     name    |               nova               |
	|     type    |             compute              |
	+-------------+----------------------------------+

	$ keystone endpoint-create --service-id=96b9ce17137744e58094009ac23f1d63 \
		--publicurl='http://controller:8774/v2/%(tenant_id)s' \
		--internalurl='http://controller:8774/v2/%(tenant_id)s' \
		--adminurl='http://controller:8774/v2/%(tenant_id)s'
	+-------------+-----------------------------------------+
	|   Property  |                  Value                  |
	+-------------+-----------------------------------------+
	|   adminurl  | http://controller:8774/v2/%(tenant_id)s |
	|      id     |     6d78f648a5a54c3dbb745480b9faa241    |
	| internalurl | http://controller:8774/v2/%(tenant_id)s |
	|  publicurl  | http://controller:8774/v2/%(tenant_id)s |
	|    region   |                regionOne                |
	|  service_id |     96b9ce17137744e58094009ac23f1d63    |
	+-------------+-----------------------------------------+

安装`noVNC`

	# commit 75d69b9f621606c3b2db48e4778ff41307f65c6d
	git clone https://github.com/kanaka/noVNC.git
	
	mkdir -p /usr/share/novnc
	cp -af noVNC/* /usr/share/novnc/

启动服务

	nova-api
	nova-cert
	nova-consoleauth
	nova-scheduler
	nova-conductor
	nova-novncproxy

验证一下

	$ nova-manage service list
	Binary           Host                                 Zone             Status     State Updated_At
	nova-cert        controller                           internal         enabled    :-)   2013-11-10 08:12:38
	nova-consoleauth controller                           internal         enabled    :-)   2013-11-10 08:12:43
	nova-scheduler   controller                           internal         enabled    :-)   2013-11-10 08:12:36
	nova-conductor   controller                           internal         enabled    :-)   2013-11-10 08:12:38
	
	$ nova-manage version
	2014.1

	$ nova image-list
	+--------------------------------------+--------------+--------+--------+
	| ID                                   | Name         | Status | Server |
	+--------------------------------------+--------------+--------+--------+
	| 4d1a5832-f229-4697-a889-39311f2611ef | Cirros 0.3.0 | ACTIVE |        |
	+--------------------------------------+--------------+--------+--------+
	
	$ glance image-list
	+--------------------------------------+--------------+-------------+------------------+---------+--------+
	| ID                                   | Name         | Disk Format | Container Format | Size    | Status |
	+--------------------------------------+--------------+-------------+------------------+---------+--------+
	| 4d1a5832-f229-4697-a889-39311f2611ef | Cirros 0.3.0 | qcow2       | bare             | 9761280 | active |
	+--------------------------------------+--------------+-------------+------------------+---------+--------+

###Compute Node

安装依赖

	apt-get install python-libvirt guestmount

修改`/etc/nova/nova.conf`

	[DEFAULT]

	sql_connection = mysql://nova:nova_password@controller/nova

	my_ip=192.168.1.11
	vncserver_listen=0.0.0.0
	vncserver_proxyclient_address=10.0.0.11

	glance_host=controller

	libvirt_type=qemu
	compute_driver=libvirt.LibvirtDriver

	state_path=/var/lib/nova
	lock_path=/var/lock/nova

	# NETWORK
	network_manager=nova.network.manager.FlatDHCPManager
	firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
	share_dhcp_address=True
	force_dhcp_release=True
	network_size=254
	allow_same_net_traffic=False
	send_arp_for_ha=True
	multi_host=True
	public_interface=eth0
	flat_interface=eth1
	flat_network_bridge=br100
	dhcpbridge_flagfile=/etc/nova/nova.conf

	# RABBITMQ
	rabbit_host=controller
	rabbit_port=5672
	rabbit_userid=guest
	rabbit_password=321321
修改`/etc/nova/api-paste.ini`

	[filter:authtoken]
	auth_host = controller
	admin_tenant_name = service
	admin_user = nova
	admin_password = nova_password

创建目录

	mkdir -p /var/lib/nova/instances

启动服务

	nova-compute
	nova-network


###配置网络

在任意Node上导入环境变量，执行`nova`命令即可进行全局设置

	$ source keystonerc_admin

创建虚机使用的网络

	$ nova network-create vmnet --fixed-range-v4=99.0.0.0/24 --bridge-interface=br100 --multi-host=T

	$ nova network-list
	+--------------------------------------+-------+-------------+
	| ID                                   | Label | Cidr        |
	+--------------------------------------+-------+-------------+
	| dfd95c9a-4887-42f5-9fa3-fd0e413dd0f4 | vmnet | 99.0.0.0/24 |
	+--------------------------------------+-------+-------------+

注入SSH公钥到虚机并确认（需要虚机Image支持）

	$ ssh-keygen      # 一路回车
	$ nova keypair-add --pub-key ~/.ssh/id_rsa.pub mykey
	
	$ nova keypair-list
	+-------+-------------------------------------------------+
	| Name  | Fingerprint                                     |
	+-------+-------------------------------------------------+
	| mykey | 1d:99:5c:70:02:a2:c2:28:cb:be:11:4b:54:51:cb:ae |
	+-------+-------------------------------------------------+

放开SSH和ICMP（Ping）的访问限制

	$ nova secgroup-list
	+----+---------+-------------+
	| Id | Name    | Description |
	+----+---------+-------------+
	| 1  | default | default     |
	+----+---------+-------------+
	
	$ nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
	$ nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
	
	$ nova secgroup-list-rules default
	+-------------+-----------+---------+-----------+--------------+
	| IP Protocol | From Port | To Port | IP Range  | Source Group |
	+-------------+-----------+---------+-----------+--------------+
	| tcp         | 22        | 22      | 0.0.0.0/0 |              |
	| icmp        | -1        | -1      | 0.0.0.0/0 |              |
	+-------------+-----------+---------+-----------+--------------+
	
确认允许IPv4转发

	$ sysctl net.ipv4.ip_forward
	net.ipv4.ip_forward = 0

	$ sysctl -w net.ipv4.ip_forward=1


###创建虚机

	$ nova secgroup-list
	+----+---------+-------------+
	| Id | Name    | Description |
	+----+---------+-------------+
	| 1  | default | default     |
	+----+---------+-------------+

	$ nova flavor-list 
	+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
	| ID | Name      | Memory_MB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public |
	+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+
	| 1  | m1.tiny   | 512       | 1    | 0         |      | 1     | 1.0         | True      |
	| 2  | m1.small  | 2048      | 20   | 0         |      | 1     | 1.0         | True      |
	| 3  | m1.medium | 4096      | 40   | 0         |      | 2     | 1.0         | True      |
	| 4  | m1.large  | 8192      | 80   | 0         |      | 4     | 1.0         | True      |
	| 5  | m1.xlarge | 16384     | 160  | 0         |      | 8     | 1.0         | True      |
	+----+-----------+-----------+------+-----------+------+-------+-------------+-----------+

	$ nova image-list
	+--------------------------------------+--------------+--------+--------+
	| ID                                   | Name         | Status | Server |
	+--------------------------------------+--------------+--------+--------+
	| 5ef6d78b-dd3e-4575-ad52-692552f3ddd3 | Cirros 0.3.0 | ACTIVE |        |
	+--------------------------------------+--------------+--------+--------+

	$ nova boot --flavor=1 --key_name=mykey --image=5ef6d78b-dd3e-4575-ad52-692552f3ddd3 --security_group=default cirros


查看虚机状态(启动需要一些时间)

	$ nova list	
	+--------------------------------------+--------+--------+------------+-------------+-------------------+
	| ID                                   | Name   | Status | Task State | Power State | Networks          |
	+--------------------------------------+--------+--------+------------+-------------+-------------------+
	| 2c0c5c9b-2511-4616-8186-3843b0800da1 | cirros | BUILD  | spawning   | NOSTATE     | private=99.0.0.2  |
	+--------------------------------------+--------+--------+------------+-------------+-------------------+
	
	$ nova list	
	+--------------------------------------+--------+--------+------------+-------------+-------------------+
	| ID                                   | Name   | Status | Task State | Power State | Networks          |
	+--------------------------------------+--------+--------+------------+-------------+-------------------+
	| 2c0c5c9b-2511-4616-8186-3843b0800da1 | cirros | ACTIVE | None       | Running     | private=99.0.0.2  |
	+--------------------------------------+--------+--------+------------+-------------+-------------------+

查看引导信息（如果启动失败，能看到报错信息）

	$ nova console-log cirros
	... （此处省略N行）
	Oct 17 10:02:00 cirros kern.info kernel: [    6.980753] ip_tables: (C) 2000-2006 Netfilter Core Team
	Oct 17 10:02:13 cirros kern.debug kernel: [   19.977012] eth0: no IPv6 routers present
	Oct 17 10:03:29 cirros authpriv.info dropbear[301]: Running in background
	############ debug end   ##############
	  ____               ____  ____
	 / __/ __ ____ ____ / __ \/ __/
	/ /__ / // __// __// /_/ /\ \ 
	\___//_//_/  /_/   \____/___/ 
	   http://cirros-cloud.net


	login as 'cirros' user. default password: 'cubswin:)'. use 'sudo' for root.
	cirros login: 

虚机应该可以被Ping通，然后SSH登陆（密码：`cubswin:)`），最后从虚机应该可以Ping通公网IP（如：`8.8.8.8`）

	ssh cirros@99.0.0.2

> 如果网络有任何异常，首先重启本机的`nova-network`和`nova-compute`服务，甚至重启系统，再作调试；重启后需要等待一会儿，等nova完成网络配置。  
> 通过`ifconfig`可以看到，增加了1个`br100`（通过`ip a`可看到和它`eth1`桥接了）和为每个虚机增加一个`vnet`（`brctl show`可看到所有桥接在一起的设备）



##安装Horizon

> 因为之前将`noVNC`安装在了`Controller Node`上，所以这里将`Horizon`也安装在它上面

获取源码

	# commit ae6abf715701b7ce026efb16c7af20b16cc90ee2 
	git clone https://github.com/openstack/horizon.git

安装

	cd horizon
	pip install -r requirements.txt
	python setup.py install
	cd ..

创建配置文件

	cd horizon/openstack_dashboard/local
	cp local_settings.py.example local_settings.py
	cd -

修改`/etc/openstack-dashboard/local_settings.py`（指定Identity Service所在的机器）

	￼OPENSTACK_HOST = "controller"

创建默认角色`Member`

	keystone role-create --name Member

启动（使用空闲端口）

	horizon/manage.py runserver 0.0.0.0:8888

访问页面

* http://localhost:8888/  （登录：`admin`,`admin_password`）

###通过Web访问`VNC Console`

要想Horizon的Console页面正常显示，需要`noVNC`，原理如下：

	vnc.html ---> nova-novncproxy(6080) ---> vnc server(5900)

安装

	# commit 75d69b9f621606c3b2db48e4778ff41307f65c6d
	git clone https://github.com/kanaka/noVNC.git
	
	mkdir -p /usr/share/novnc
	cp -af noVNC/* /usr/share/novnc/

确认启动服务

	nova-novncproxy
	nova-consoleauth

> 如果不将noVNC安装到系统中，也可以如下方式启动：
> 
> 		noVNC/utils/nova-novncproxy --config-file /etc/nova/nova.conf --web `pwd`/noVNC/

###通过Client访问`VNC Console`

工作原理
	
	xvpvncviewer ---> nova-xvpvncproxy(6081) ---> vnc server(5900)

安装

	# commit fc292084732bd20bc69746a0567001293b63608f
	git clone https://github.com/cloudbuilders/nova-xvpvncviewer

	cd nova-xvpvncviewer/viewer
	make

确认启动服务

	nova-xvpvncproxy
	nova-consoleauth

手动获取URL

	nova get-vnc-console [VM_ID or VM_NAME] xvpvnc

启动客户端

	java -jar VncViewer.jar url [URL]

###使用Apache提供Web服务

见官方文档



## 安装Swift

获取源码

	# commit f46b48e1dc5747faef5857d331db9a441b8b6af6
	git clone git://github.com/openstack/swift.git

安装依赖

	cd swift
	pip install -r requirements.txt
	apt-get install memcached
	
> pip安装依赖时可能报错`error: ffi.h: No such file or directory`，见`Troubleshooting`部分
	
安装swift到系统

	python setup.py install
	
### 创建Ring Files

Ring File包含存储设备的所有信息 （each ring will contain 2^12=4096 partitions）

	mkdir -pv /etc/swift
	swift-ring-builder /etc/swift/account.builder create 12 3 1
	swift-ring-builder /etc/swift/container.builder create 12 3 1
	swift-ring-builder /etc/swift/object.builder create 12 3 1

为每个Ring增加存储设备

	swift-ring-builder /etc/swift/account.builder add z1-127.0.0.1:6002/z1d1 100
	swift-ring-builder /etc/swift/account.builder add z1-127.0.0.1:6002/z1d2 100
	swift-ring-builder /etc/swift/account.builder add z2-127.0.0.1:6002/z2d1 100
	swift-ring-builder /etc/swift/account.builder add z2-127.0.0.1:6002/z2d2 100
	swift-ring-builder /etc/swift/account.builder add z3-127.0.0.1:6002/z3d1 100
	swift-ring-builder /etc/swift/account.builder add z3-127.0.0.1:6002/z3d2 100

	swift-ring-builder /etc/swift/container.builder add z1-127.0.0.1:6001/z1d1 100
	swift-ring-builder /etc/swift/container.builder add z1-127.0.0.1:6001/z1d2 100
	swift-ring-builder /etc/swift/container.builder add z2-127.0.0.1:6001/z2d1 100
	swift-ring-builder /etc/swift/container.builder add z2-127.0.0.1:6001/z2d2 100
	swift-ring-builder /etc/swift/container.builder add z3-127.0.0.1:6001/z3d1 100
	swift-ring-builder /etc/swift/container.builder add z3-127.0.0.1:6001/z3d2 100
	
	swift-ring-builder /etc/swift/object.builder add z1-127.0.0.1:6000/z1d1 100
	swift-ring-builder /etc/swift/object.builder add z1-127.0.0.1:6000/z1d2 100
	swift-ring-builder /etc/swift/object.builder add z2-127.0.0.1:6000/z2d1 100
	swift-ring-builder /etc/swift/object.builder add z2-127.0.0.1:6000/z2d2 100
	swift-ring-builder /etc/swift/object.builder add z3-127.0.0.1:6000/z3d1 100
	swift-ring-builder /etc/swift/object.builder add z3-127.0.0.1:6000/z3d2 100
	
To distribute the partitions across the drives in the ring

	swift-ring-builder /etc/swift/account.builder rebalance
	swift-ring-builder /etc/swift/container.builder rebalance
	swift-ring-builder /etc/swift/object.builder rebalance
	
检查文件是否生成

	ls /etc/swift/*gz
	/etc/swift/account.ring.gz  /etc/swift/container.ring.gz  /etc/swift/object.ring.gz
	
为swift创建HashKey (使用`openssl rand -hex 10`生成随机串`0b1b9109c1ddde4f9c4b`)

	cat > /etc/swift/swift.conf << EOF
	[swift-hash]
	swift_hash_path_suffix = 0b1b9109c1ddde4f9c4b
	EOF

	chmod 600 /etc/swift/swift.conf


###配置Swift Proxy

增加配置文件`/etc/swift/proxy-server.conf`

	tee /etc/swift/proxy-server.conf <<EOF
	[DEFAULT]
	bind_port = 8080
	workers = 3
	user = swift

	[pipeline:main]
	pipeline = healthcheck cache authtoken keystone proxy-server

	[app:proxy-server]
	use = egg:swift#proxy
	allow_account_management = true
	account_autocreate = true

	[filter:cache]
	use = egg:swift#memcache
	memcache_servers = 127.0.0.1:11211

	[filter:catch_errors]
	use = egg:swift#catch_errors

	[filter:healthcheck]
	use = egg:swift#healthcheck

	[filter:keystone]
	#paste.filter_factory = keystone.middleware.swift_auth:filter_factory
	use = egg:swift#keystoneauth
	operator_roles = admin, SwiftOperator
	is_admin = true
	cache = swift.cache

	[filter:authtoken]
	paste.filter_factory = keystone.middleware.auth_token:filter_factory
	auth_host = 127.0.0.1
	auth_port = 35357
	auth_protocol = http
	auth_uri = http://127.0.0.1:35357
	# if its defined
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456
	EOF

然后

	groupadd swift
	useradd -g swift swift
	chown -R swift:swift /etc/swift/
	
	LANG=en_US.utf8
	service memcached start

###配置Keystone

切换到管理员

	. ~/keystonerc_admin
	
创建用户、租户（admin角色前面已经创建）

	keystone tenant-create --name services
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |                                  |
	|   enabled   |               True               |
	|      id     | 750f121b2087434e91393d554a5940d4 |
	|     name    |             services             |
	+-------------+----------------------------------+

	keystone user-create --name swift --pass 789789
	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|  email   |                                  |
	| enabled  |               True               |
	|    id    | e981e5f5a4244456b3a3cfdb52cd9707 |
	|   name   |              swift               |
	| tenantId |                                  |
	+----------+----------------------------------+

将swift用户设置为admin角色和services租户成员

	keystone role-list
	+----------------------------------+----------+
	|                id                |   name   |
	+----------------------------------+----------+
	| 9fe2ff9ee4384b1894a90878d3e92bab | _member_ |
	| de5cdd28a71f4df2943ac617ed20695c |  admin   |
	| 430f055d142649d1b02e685025b07a5b |   user   |
	+----------------------------------+----------+
	
	keystone user-role-add --role de5cdd28a71f4df2943ac617ed20695c \
	--tenant_id 750f121b2087434e91393d554a5940d4 \
	--user e981e5f5a4244456b3a3cfdb52cd9707
	
创建service和endpoint

	keystone service-create --name swift --type object-store \
	--description "Swift Storage Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |      Swift Storage Service       |
	|      id     | c09efaf486e54266b328cad0a52752c2 |
	|     name    |              swift               |
	|     type    |           object-store           |
	+-------------+----------------------------------+
	
	keystone endpoint-create --service_id c09efaf486e54266b328cad0a52752c2 \
	--publicurl "http://127.0.0.1:8080/v2/AUTH_\$(tenant_id)s" \
	--adminurl "http://127.0.0.1:8080/v2/AUTH_\$(tenant_id)s" \
	--internalurl "http://127.0.0.1:8080/v2/AUTH_\$(tenant_id)s"
    +-------------+---------------------------------------------+
	|   Property  |                    Value                    |
	+-------------+---------------------------------------------+
	|   adminurl  | http://127.0.0.1:8080/v2/AUTH_$(tenant_id)s |
	|      id     |       20f2279d7b2345c38f5315ac4518f1ea      |
	| internalurl | http://127.0.0.1:8080/v2/AUTH_$(tenant_id)s |
	|  publicurl  | http://127.0.0.1:8080/v2/AUTH_$(tenant_id)s |
	|    region   |                  regionOne                  |
	|  service_id |       c09efaf486e54266b328cad0a52752c2      |
	+-------------+---------------------------------------------+
	
查看一下

	keystone service-list
	+----------------------------------+----------+--------------+---------------------------+
	|                id                |   name   |     type     |        description        |
	+----------------------------------+----------+--------------+---------------------------+
	| 714af530900840ef88106765f13f1921 | keystone |   identity   | Keystone Identity Service |
	| c09efaf486e54266b328cad0a52752c2 |  swift   | object-store |   Swift Storage Service   |
	+----------------------------------+----------+--------------+---------------------------+

	keystone endpoint-list
	+----------------------------------+-----------+---------------------------------------------+---------------------------------------------+---------------------------------------------+----------------------------------+
	|                id                |   region  |                  publicurl                  |                 internalurl                 |                   adminurl                  |            service_id            |
	+----------------------------------+-----------+---------------------------------------------+---------------------------------------------+---------------------------------------------+----------------------------------+
	| 20f2279d7b2345c38f5315ac4518f1ea | regionOne | http://127.0.0.1:8080/v2/AUTH_$(tenant_id)s | http://127.0.0.1:8080/v2/AUTH_$(tenant_id)s | http://127.0.0.1:8080/v2/AUTH_$(tenant_id)s | c09efaf486e54266b328cad0a52752c2 |
	| d28d840d4be74a778578953a1a13324f | regionOne |          http://127.0.0.1:5000/v2.0         |          http://127.0.0.1:5000/v2.0         |         http://127.0.0.1:35357/v2.0         | 714af530900840ef88106765f13f1921 |
	+----------------------------------+-----------+---------------------------------------------+---------------------------------------------+---------------------------------------------+----------------------------------+
	
###配置Swift Storage Nodes

创建配置文件

`/etc/swift/account-server.conf`

	tee /etc/swift/account-server.conf <<EOF
	[DEFAULT]
	devices = /srv/node
	bind_ip = 127.0.0.1
	bind_port = 6002
	mount_check = false
	user = swift
	log_facility = LOG_LOCAL2
	workers = 3

	[pipeline:main]
	pipeline = account-server

	[app:account-server]
	use = egg:swift#account

	[account-replicator]
	concurrency = 1

	[account-auditor]

	[account-reaper]
	concurrency = 1
	EOF

`/etc/swift/container-server.conf`

	tee /etc/swift/container-server.conf <<EOF
	[DEFAULT]
	devices = /srv/node
	bind_ip = 127.0.0.1
	bind_port = 6001
	mount_check = false
	user = swift
	log_facility = LOG_LOCAL2
	workers = 3

	[pipeline:main]
	pipeline = container-server

	[app:container-server]
	use = egg:swift#container

	[container-replicator]
	concurrency = 1

	[container-updater]
	concurrency = 1

	[container-auditor]

	[container-sync]

	EOF

`/etc/swift/object-server.conf`

	tee /etc/swift/object-server.conf <<EOF
	[DEFAULT]
	devices = /srv/node
	bind_ip = 127.0.0.1
	bind_port = 6000
	mount_check = false
	user = swift
	log_facility = LOG_LOCAL2
	workers = 3

	[pipeline:main]
	pipeline = object-server

	[app:object-server]
	use = egg:swift#object

	[object-replicator]
	concurrency = 1

	[object-updater]
	concurrency = 1

	[object-auditor]
	
	[object-expirer]
	EOF
	
`/etc/swift/object-expirer.conf`

	cat > /etc/swift/object-expirer.conf << EOF
    [DEFAULT]

    [object-expirer]
    interval = 300

    [pipeline:main]
    pipeline = catch_errors cache proxy-server

    [app:proxy-server]
    use = egg:swift#proxy

    [filter:cache]
    use = egg:swift#memcache

    [filter:catch_errors]
    use = egg:swift#catch_errors
    EOF

创建和挂载loopback设备（注意：系统重启后挂载失效）

	for zone in 1 2 3 ; do
		for device in 1 2 ; do
			truncate /var/tmp/swift-device-z${zone}d${device} --size 5G
			LOOPDEVICE=$(losetup --show -f /var/tmp/swift-device-z${zone}d${device})
			mkfs.ext4 -I 1024 $LOOPDEVICE
			mkdir -p /srv/node/z${zone}d${device}
			mount -o noatime,nodiratime,nobarrier,user_xattr $LOOPDEVICE \
			/srv/node/z${zone}d${device}
		done
	done
	
	chown -R swift:swift /srv/node /etc/swift
	
启动Swift

	swift-init all start
	
> 启动时可能会报错`NameError: name '_' is not defined`，解决方案见`Troubleshooting`部分

重启系统后需要重新挂载，执行

	for zone in 1 2 3 ; do
	    for device in 1 2 ; do
	        LOOPDEVICE=$(losetup --show -f /var/tmp/swift-device-z${zone}d${device})
	        mount -o noatime,nodiratime,nobarrier,user_xattr $LOOPDEVICE \
	        /srv/node/z${zone}d${device}
	    done
	done
	
###测试Swift

	. ~/keystonerc_admin
	
	swift list
	
	head -c 1024 /dev/urandom > data.file ; swift upload c1 data.file
	head -c 1024 /dev/urandom > data2.file ; swift upload c1 data2.file
	head -c 1024 /dev/urandom > data3.file ; swift upload c2 data3.file
	
	$ swift list
	c1
	c2
	
	$ swift list c1
	data.file
	data2.file
	
	$ swift list c2
	data3.file



##安装Cinder

获取源码

	# commit 300ce61120ce179d1d0e4ffe8aa0bd4ceaddba6b 
	git clone https://github.com/openstack/cinder.git

	# commit d21ed05b4ed1a5b5321876c481e0286d3696afd5 
	git clone https://github.com/openstack/python-cinderclient.git

安装

	cd cinder
	pip install -r requirements.txt
	python setup.py  install
	
	cd ../python-cinderclient/
	pip install -r requirements.txt
	python setup.py  install
	cd ..
	
	apt-get install tgt open-iscsi rabbitmq-server
	
安装配置

	cp -af cinder/etc/cinder /etc
	cp /etc/cinder/cinder.conf.sample /etc/cinder/cinder.conf
	
修改`/etc/cinder/api-paste.ini`

	[filter:authtoken]
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456
	service_protocol = http
	service_host = 127.0.0.1
	service_port = 5000

修改`/etc/cinder/cinder.conf`

	[DEFAULT]
	rootwrap_config=/etc/cinder/rootwrap.conf
	sql_connection=mysql://root:111111@127.0.0.1/cinder
	api_paste_config=/etc/cinder/api-paste.ini
	
	state_path=/var/lib/cinder
	volumes_dir=$state_path/volumes

	iscsi_helper=tgtadm
	volume_name_template=volume-%s
	volume_group=cinder-volumes
	verbose=true
	auth_strategy=keystone
	
	log_file=cinder.log
	log_dir=/var/log/cinder
	
	rabbit_host=localhost
	rabbit_port=5672
	rabbit_userid=guest
	rabbit_password=321321
	rabbit_virtual_host=/

创建cinder数据库

	mysql -u root -p
	create database cinder;
	quit

配置RabbitMQ

	rabbitmqctl change_password guest 321321

配置TGT

	mkdir -p /var/lib/cinder/volumes
	sh -c "echo 'include /var/lib/cinder/volumes/*' >> /etc/tgt/conf.d/cinder.conf"

	restart tgt

初始化数据库

	mkdir -p /var/log/cinder
	cinder-manage db sync
	
创建卷

	dd if=/dev/zero of=~/cinder-volumes bs=1 count=0 seek=2G
	losetup /dev/loop6 ~/cinder-volumes
	pvcreate /dev/loop6
	vgcreate cinder-volumes /dev/loop6
	
	$ pvscan
	PV /dev/sda5    VG precise64        lvm2 [79.76 GiB / 0    free]
	PV /dev/loop6   VG cinder-volumes   lvm2 [2.00 GiB / 2.00 GiB free]
	Total: 2 [81.75 GiB] / in use: 2 [81.75 GiB] / in no VG: 0 [0   ]

启动服务

	cinder-volume --config-file=/etc/cinder/cinder.conf
	cinder-api --config-file=/etc/cinder/cinder.conf
	cinder-scheduler --config-file=/etc/cinder/cinder.conf

创建`service`和`endpoint`

	keystone service-create --name=volume --type=volume --description="Nova Volume Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |      Nova Volume Service         |
	|      id     | 0f66c20499ca4f77990e286a3607e5aa |
	|     name    |              volume              |
	|     type    |              volume              |
	+-------------+----------------------------------+
	
	keystone endpoint-create --service_id=0f66c20499ca4f77990e286a3607e5aa \
                        --publicurl "http://localhost:8776/v1/\$(tenant_id)s" \
                        --adminurl "http://localhost:8776/v1/\$(tenant_id)s" \
                        --internalurl "http://localhost:8776/v1/\$(tenant_id)s"
	+-------------+----------------------------------------+
	|   Property  |                 Value                  |
	+-------------+----------------------------------------+
	|   adminurl  | http://localhost:8776/v1/$(tenant_id)s |
	|      id     |    a62cb598466f411bb8f6381495994690    |
	| internalurl | http://localhost:8776/v1/$(tenant_id)s |
	|  publicurl  | http://localhost:8776/v1/$(tenant_id)s |
	|    region   |               regionOne                |
	|  service_id |    0f66c20499ca4f77990e286a3607e5aa    |
	+-------------+----------------------------------------+
	
	keystone endpoint-list

创建1G的volume测试一下（取名：test）

	cinder list
	+----+--------+--------------+------+-------------+----------+-------------+
	| ID | Status | Display Name | Size | Volume Type | Bootable | Attached to |
	+----+--------+--------------+------+-------------+----------+-------------+
	+----+--------+--------------+------+-------------+----------+-------------+

	cinder create --display_name test 1
	+---------------------+--------------------------------------+
	|       Property      |                Value                 |
	+---------------------+--------------------------------------+
	|     attachments     |                  []                  |
	|  availability_zone  |                 nova                 |
	|       bootable      |                False                 |
	|      created_at     |      2013-09-23T13:08:54.358719      |
	| display_description |                 None                 |
	|     display_name    |                 test                 |
	|          id         | 7e7a503c-0f47-4a74-88a6-e4fd84a9592b |
	|       metadata      |                  {}                  |
	|         size        |                  1                   |
	|     snapshot_id     |                 None                 |
	|     source_volid    |                 None                 |
	|        status       |               creating               |
	|     volume_type     |                 None                 |
	+---------------------+--------------------------------------+

	cinder list
	+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
	|                  ID                  |   Status  | Display Name | Size | Volume Type | Bootable | Attached to |
	+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+
	| 7e7a503c-0f47-4a74-88a6-e4fd84a9592b | available |     test     |  1   |     None    |  False   |             |
	+--------------------------------------+-----------+--------------+------+-------------+----------+-------------+




#Troubleshooting

## Expecting an auth URL via either --os-auth-url or env[OS_AUTH_URL]

缺少环境变量，使用`admin_token`验证方式设置如下（`SERVICE_TOKEN`的值取自`/etc/keystone/keystone.conf`的`admin_token`字段）

    export SERVICE_TOKEN=ADMIN
    export SERVICE_ENDPOINT=http://127.0.0.1:35357/v2.0/
    
或者创建用户的环境变量快速设置脚本，并运行

	cat > ~/keystonerc_joe << EOF
	export OS_USERNAME=joe
	export OS_TENANT_NAME=trial
	export OS_PASSWORD=123123
	export OS_AUTH_URL=http://127.0.0.1:5000/v2.0/
	export PS1="[\u@\h \W(keystone_joe)]\$ "
	EOF
	
	. ~/keystonerc_joe

注：管理员端口为`35357`，普通用户是`5000`，参考`/etc/keystone/keystone.conf`的配置


## Unable to authorize user

可能是数据库中不存在`identity`这个`endpoint`, 且`/etc/keystone/keystone.conf`又配置为从数据读取

	[catalog]
	driver = keystone.catalog.backends.sql.Catalog
	# driver = keystone.catalog.backends.templated.TemplatedCatalog
	# template_file = default_catalog.templates

可配置为从模板读取

    [catalog]
    # driver = keystone.catalog.backends.sql.Catalog
    driver = keystone.catalog.backends.templated.TemplatedCatalog
    template_file = default_catalog.templates
    
如果一定要从数据库读取，先增加`admin_token`方式验证的环境变量，然后手动增加endpoint信息，可参考`/etc/keystone/default_catalog.templates`

## [swift, pip install -r reuqirements.txt] ERROR： "c/_cffi_backend.c:14:17: fatal error: ffi.h: No such file or directory"

注释掉`requirements.txt`中的`xattr`,安装完依赖后自己手动安装`python-xattr`包

    pip install -r requirements.txt
    apt-get install python-xattr

## [swift-init all start] ERROR: "NameError: name '_' is not defined"

`/usr/local/lib/python2.7/dist-packages/keystone/exception.py` 增加：

    import gettext
    _ = gettext.gettext

## No module named swift_auth

`/etc/swift/proxy-server.conf` 修改：

	[filter:keystone]
	# paste.filter_factory = keystone.middleware.swift_auth:filter_factory
    use = egg:swift#keystoneauth

## [swift list] Endpoint for object-store not found

如果`/etc/keystone/proxy-server.conf`配置为从模版读取,那么必须手动添加`endpoint`到模版，数据库创建语句无效。

    [catalog]
    # dynamic, sql-based backend (supports API/CLI-based management commands)
    #driver = keystone.catalog.backends.sql.Catalog

    # static, file-based backend (does *NOT* support any management commands)
    driver = keystone.catalog.backends.templated.TemplatedCatalog

    template_file = default_catalog.templates

`/etc/keystone/default_catalog.templates` 增加:

    catalog.RegionOne.object-store.publicURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
    catalog.RegionOne.object-store.adminURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
    catalog.RegionOne.object-store.internalURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
    catalog.RegionOne.object-store.name = Object Store Service

## [swift-init all start] Unable to locate config for object-expirer

`/etc/swift/object-server.conf` 增加：

    [object-expirer]

新建`/etc/swift/object-expirer.conf`

	cat > /etc/swift/object-expirer.conf << EOF
    [DEFAULT]

    [object-expirer]
    interval = 300

    [pipeline:main]
    pipeline = catch_errors cache proxy-server

    [app:proxy-server]
    use = egg:swift#proxy

    [filter:cache]
    use = egg:swift#memcache

    [filter:catch_errors]
    use = egg:swift#catch_errors
    EOF

设置权限

	chown -R swift:swift /etc/swift
    
## [swift-init all start] Unable to locate config for proxy-server

缺少`/etc/swift/proxy-server.conf`文件
    
    

## [Log] No such file or directory: '/var/cache/swift/object.recon'

`proxy-server.conf`增加：

    [filter:recon]
    use = egg:swift#recon
    recon_cache_path = /var/cache/swift

    mkdir /var/cache/swift
    chmod 777 /var/cache/swift

## [glance-control api start] \__init__() got an unexpected keyword argument 'parents'

    pip freeze | grep oslo
    # oslo.config==1.2.0a3
    
    pip uninstall oslo.config
    pip install oslo.config==1.1.0

## 没有glance命令

`glance`命令在包`glanceclient`中，该包是独立的，可以直接用`pip`快速安装：

    pip install python-glanceclient

## [glance add] Authorization Failed: *** (HTTP Unable to establish connection to http://127.0.0.1:35357/v2.0/tokens)

确认keystone服务是否启动

    ps -Af | grep keystone
    keystone-all
    
## [glance add] Authorization Failed: *** (HTTP Unable to establish connection to https://127.0.0.1:35357/v2.0/tokens)

注意，这里的报错是`https`，检查配置文件`/etc/glance/glance-api.conf`

	# Valid schemes are 'http://' and 'https://'
	# If no scheme specified,  default to 'https://'
	swift_store_auth_address = http://127.0.0.1:35357/v2.0/
	
注意注释部分，如果没有`http://`则默认使用`https://`


## [glance add] code 400, message Bad HTTP/0.9 request type

启动命令换成如下，可看到日志错误提示信息：

    glance-api --config-file /etc/glance/glance-api.conf
    glance-registry --config-file /etc/glance/glance-registry.conf

## [swift upload] 404 Not Found. The resource could not be found.

`chown -R swift:swift /srv/node`  (目录写权限问题)

## [glance index] WARNING keystone.common.controller [-] RBAC: Bypassing authorization

可能是用admin验证引起的（启动方式`glance-control api/registry start`）

也可能是启动方式问题，使用`glance-api`和`glance-registry`启动后消失

## [glance add, /var/log/glance/api.log] Container HEAD failed: http://localhost:8080/v1/AUTH_9cb8bbc75bdb4484af790cfc3e4343e5/glance 404 Not Found

swift中没有`glance`这个container，可以手动创建，也可以修改配置`/etc/glance/glance-api.conf`

    swift_store_container = glance

    # Do we create the container if it does not exist?
    swift_store_create_container_on_put = True

## [glance-api --config-file /etc/glance/glance-api.conf] Stderr: '/bin/sh: 1: collie: not found\n'

	apt-get install sheepdog
	
## [glance add] ClientException: Unauthorised. Check username, password and tenant name/id

账户设置有问题，可以检查配置`/etc/glance/glance-api.conf`，也可以在报错文件的语句前插入打印语句查看

	swift_store_auth_address = http://127.0.0.1:35357/v2.0/
	swift_store_user = admin:admin
	swift_store_key = 123456

## [glance image-list] HTTPInternalServerError (HTTP 500)

- 确认安装`curl`。加上`-d`参数，可看到执行了`curl`命令，手动执行后报错`The program 'curl' is currently not installed.`
- 确认`glance db_sync`执行成功。查看数据库`glance`，发现没有表，检查`glance-*.conf`的`sql-connection`配置
- 确认`glance-api.conf`文件中的`swift_store_auth_address = http://127.0.0.1:35357/v2.0/`是`http`

## [nova, pvcreate /dev/loop2] Device /dev/loop2 not found (or ignored by filtering)

可能是修改了`/etc/lvm/lvm.conf`中的`filter`造成的

## [cinder create] ERROR cinder.scheduler.filters.capacity_filter [***] Free capacity not set: volume node info collection broken.
## [cinder create] ERROR cinder.volume.flows.create_volume [***] Failed to schedule_create_volume: No valid host was found.

使用`cinder-all`启动所有服务，然后`cinder create`时会就遇到这个报错；

使用如下启动方式时正常：

	cinder-volume --config-file=/etc/cinder/cinder.conf
	cinder-api --config-file=/etc/cinder/cinder.conf
	cinder-scheduler  --config-file=/etc/cinder/cinder.conf

## [cinder list] `Status`一直都是`creating`

	cinder list
	+--------------------------------------+----------+--------------+------+-------------+----------+-------------+
	|                  ID                  |  Status  | Display Name | Size | Volume Type | Bootable | Attached to |
	+--------------------------------------+----------+--------------+------+-------------+----------+-------------+
	| fbda9189-26e2-4e42-9622-f707be57d565 | creating |     test     |  1   |     None    |  False   |             |
	+--------------------------------------+----------+--------------+------+-------------+----------+-------------+

首先确定数据库`cinder`中是否有表，即确保`cinder-manage db sync`执行成功

## [cinder create] `Status`是`error`，`cinder-volume`报错：`Exit code: 5`

确定配置文件`/etc/cinder/cinder.conf`包含如下内容

	state_path=/var/lib/cinder
	volumes_dir=$state_path/volumes
	
## [cinder-volume] AMQP server on localhost:5672 is unreachable

在不使用nova的情况下，确认`/etc/cinder/cinder.conf`

	rabbit_virtual_host=/
	
## [nova-compute] ImportError: No module named libvirt

	apt-get install python-libvirt

## [nova-api] Address already in use

先确定是否已经有`nova-api`进程在运行

	ps -ef | grep nova
	
如果没有，杀掉占用`8777`的进程

	netstat -apn | grep 8777
	
## [nova image-list] ERROR: printt

`novaclient`代码有bug，修改`/usr/lib/python2.7/dist-packages/novaclient/utils.py`

	def print_list(objs, fields, formatters={}):
	    //此处省略N行
	    print pt.get_string(sortby=fields[0])
        #pt.printt(sortby=fields[0])
        
安装`novnc`会作为依赖安装`novaclient`包，如果比较早的版本可能会有问题，用如下方式安装

	apt-get download novnc
	dpkg --force-all -i novnc_*.deb
        
## [nova image-list] ERROR: Unauthorized (HTTP 401)

修改`/etc/nova/api-paste.ini`

	[filter:authtoken]
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456

## [nova net-list] ERROR: HTTPConnectionPool(host='10.0.2.15', port=8774): Max retries exceeded with url: /v2/dd3d73c9f6e64acca28376d9bad0fc58/os-tenant-networks (Caused by <class 'socket.error'>: [Errno 111] Connection refused)

使用`devstack`在虚拟机安装后，执行`nova net-list`报错，执行`netstat`查看`8774`端口没有被监听，然后发现`nova-api`没有起来，手动启动后报错`OSError: [Errno 12] Cannot allocate memory`，当前虚拟及内存只分配了1G，扩大到2G后正常。

## [nova-compute] libvirtError: internal error Cannot find suitable emulator for x86_64

	apt-get install guestmount
	
## [nova-network] nova.openstack.common.rpc.amqp OSError: [Errno 2] No such file or directory

* 确定有该命令：`dhcp_release`，安装包`dnsmasq-utils`

## [Ping] 虚机不能被Ping通

* 确认打开SSH和ICMP访问限制
* 确认打开ip_v4转发
* 确认`my_ip`设为能访问外网网卡的IP
* 参考链接：<https://ask.openstack.org/en/question/120/cantt-ping-my-vm-from-controller-node/>

## [创建项目] NotFound: ***"Member"

Horizon的“项目”页面点击“创建项目”按钮报错，因为缺少默认角色`Member`，创建即可

	keystone role-create --name Member

## [nova-consoleauth] UnsupportedRpcVersion: Specified RPC version, 1.0, not supported by this endpoint.

Horizon的Console页面不显示，发现`nova-consoleauth`报错  
`noVNC`给`nova-consoleauth`发了一个`check_token`的RPC，RPC版本为空则默认设为1.0，而`nova-consoleauth`这边默认是2.0的API，在检查API版本兼容性时报错，可以硬改成2.0，也能过；  
但是最后页面还是显示不出来，用xvpvncviewer能接上，但看不到任何东西，暂不能确定问题所在。

## [keystone] fatal error: Python.h: No such file or directory

	apt-get install python-dev

## [keystone] fatal error: libxml/xmlversion.h: No such file or directory

	apt-get install libxml2-dev libxslt-dev

## [keystone] Authorization Failed: Unable to sign token. (HTTP 500)

执行`keystone`命令报错，查看日志报错如下：

	 Command 'openssl' returned non-zero exit status 3

需要初始化证书

	keystone-manage pki_setup --keystone-user=root --keystone-group=root

## [nova-novncproxy] Can not find novnc html/js/css files at /usr/share/novnc.

	git clone https://github.com/kanaka/noVNC.git
	mkdir -p /usr/share/novnc
	cp -af noVNC/* /usr/share/novnc/
	nova-novncproxy

## [nova-compute] ERROR nova.virt.driver [-] Compute driver option required, but not specified

`/etc/nova/nova.conf`

	libvirt_type=qemu
	compute_driver=libvirt.LibvirtDriver

## [nova-compute] No such file or directory: '/usr/local/lib/python2.7/dist-packages/instances'

`/etc/nova/nova.conf`

	state_path=/var/lib/nova

然后

	mkdir -p /var/lib/nova/instances
	nova-compute

## [nova-compute] libvirtError: internal error Cannot find suitable emulator for i686

	apt-get install guestmount

## [nova-network] Failed to read some config files: /etc/nova/nova-dhcpbridge.conf

`/etc/nova/nova.conf`

	dhcpbridge_flagfile=/etc/nova/nova.conf

## [git clone] error: Couldn't resolve host 'github.com' while accessing https://github.com/openstack/horizon.git/info/refs

没有DNS，在`/etc/resolv.conf`增加：

	nameserver 8.8.8.8