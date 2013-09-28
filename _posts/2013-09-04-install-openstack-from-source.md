# 声明

# 安装

## 准备实验环境

本文使用`Vagrant`虚拟机（VirtualBox的一个前端），安装`Ubuntu 12.04（64-bit）`作为实验环境

先在Host机上安装好`VirtualBox`和`Vagrant`	

下载虚机映象

	vagrant box add precise64 http://files.vagrantup.com/precise64.box

创建配置文件（自定义工作目录：openstack）

	mkdir ~/openstack
	cd openstack
	vagrant init precise64

增加如下内容到新生成的配置文件（Host机通过`8080`端口访问虚机的80端口；2G虚拟内存）

	config.vm.network :forwarded_port, guest: 80, host: 8080
	config.vm.provider :virtualbox do |vb|
		vb.customize ["modifyvm", :id, "--memory", "2048"]
	end

启动虚机并SSH登陆

	vagrant up
	vagrant ssh
	
国内用户可以换更快的源

	sudo su
	cat > /etc/apt/sources.list << EOF
	deb http://mirrors.163.com/ubuntu/ precise main universe restricted multiverse
	deb-src http://mirrors.163.com/ubuntu/ precise main universe restricted multiverse
	deb http://mirrors.163.com/ubuntu/ precise-security universe main multiverse restricted
	deb-src http://mirrors.163.com/ubuntu/ precise-security universe main multiverse restricted
	deb http://mirrors.163.com/ubuntu/ precise-updates universe main multiverse restricted
	deb http://mirrors.163.com/ubuntu/ precise-proposed universe main multiverse restricted
	deb-src http://mirrors.163.com/ubuntu/ precise-proposed universe main multiverse restricted
	deb http://mirrors.163.com/ubuntu/ precise-backports universe main multiverse restricted
	deb-src http://mirrors.163.com/ubuntu/ precise-backports universe main multiverse restricted
	deb-src http://mirrors.163.com/ubuntu/ precise-updates universe main multiverse restricted
	EOF
	
	apt-get update
	
安装一些基本工具

	apt-get install vim build-essential git python-dev python-setuptools python-pip libxml2-dev libxslt-dev
	
附：当工作未完成且需要关机时，先“保存状态后关闭”，以后再“从保存状态启动”

	vagrant suspend
	vagrant up
	
## 安装数据库

	sudo apt-get install mysql-server mysql-client python-mysqldb

终端弹出界面，提示输入数据库的root用户密码（例如：`111111`）

## 安装Keystone

创建keystone数据库

	mysql -u root -p
	create database keystone;
	quit

获取源码

	git clone git://github.com/openstack/keystone.git
	
安装依赖

	cd keystone
	pip install -r requirements.txt
	
安装keystone到系统

	python setup.py install
	
复制配置文件

	mkdir -p /etc/keystone
	cp etc/* /etc/keystone/
	
	cd /etc/keystone
	cp keystone.conf.sample keystone.conf
	
修改`/etc/keystone/keystone.conf` (使用`openssl rand -hex 10`生成`token：fa9a647b8de836869722`随机串)

	[DEFAULT]
	admin_token = fa9a647b8de836869722
	public_port = 5000
	admin_port = 35357
	public_endpoint = http://localhost:%(public_port)s/v2.0
	admin_endpoint = http://localhost:%(admin_port)s/v2.0

	[sql]
	# connection = sqlite:///keystone.db
	connection = mysql://root:111111@localhost/keystone

	[catalog]
	driver = keystone.catalog.backends.sql.Catalog
	# driver = keystone.catalog.backends.templated.TemplatedCatalog
	# template_file = default_catalog.templates

初始化数据库

	keystone-manage db_sync
	
初始化证书 （参数意义：`chmod root:root`）

	keystone-manage pki_setup --keystone-user=root --keystone-group=root
	
启动keystone服务

	keystone-all -d &
	
设置环境变量（目前还没有账户，只能先使用`admin_token`方式验证）

	export SERVICE_TOKEN=fa9a647b8de836869722
	export SERVICE_ENDPOINT=http://localhost:35357/v2.0
	
创建类型为`identity`的`service`和相应的`endpoint`

	keystone service-create --name=keystone --type=identity \
	--description="Keystone Identity Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |    Keystone Identity Service     |
	|      id     | 714af530900840ef88106765f13f1921 |
	|     name    |             keystone             |
	|     type    |             identity             |
	+-------------+----------------------------------+
	
	keystone endpoint-create --service_id 714af530900840ef88106765f13f1921 \
	--publicurl 'http://127.0.0.1:5000/v2.0' \
	--adminurl 'http://127.0.0.1:35357/v2.0' \
	--internalurl 'http://127.0.0.1:5000/v2.0'
	
	keystone endpoint-list
	+----------------------------------+-----------+----------------------------+----------------------------+-----------------------------+----------------------------------+
	|                id                |   region  |         publicurl          |        internalurl         |           adminurl          |            service_id            |
	+----------------------------------+-----------+----------------------------+----------------------------+-----------------------------+----------------------------------+
	| d28d840d4be74a778578953a1a13324f | RegionOne | http://127.0.0.1:5000/v2.0 | http://127.0.0.1:5000/v2.0 | http://127.0.0.1:35357/v2.0 | 714af530900840ef88106765f13f1921 |
	+----------------------------------+-----------+----------------------------+----------------------------+-----------------------------+----------------------------------+
	
### 创建管理员（admin）账户

创建用户、角色、租户

	keystone user-create --name admin --pass 123456
	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|  email   |                                  |
	| enabled  |               True               |
	|    id    | 94d416d8ebf34d3e97e345afcc5a2283 |
	|   name   |              admin               |
	| tenantId |                                  |
	+----------+----------------------------------+
	
	keystone role-create --name admin
	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|    id    | de5cdd28a71f4df2943ac617ed20695c |
	|   name   |              admin               |
	+----------+----------------------------------+

	keystone tenant-create --name admin
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |                                  |
	|   enabled   |               True               |
	|      id     | 40c76f9aa1c44907aa1a68b9cd7e8034 |
	|     name    |              admin               |
	+-------------+----------------------------------+

将admin用户设置为admin角色和admin租户的成员 （参数为上面动态生成的ID）

	keystone user-role-add --user 94d416d8ebf34d3e97e345afcc5a2283 \
	--role de5cdd28a71f4df2943ac617ed20695c \
	--tenant_id 40c76f9aa1c44907aa1a68b9cd7e8034
	
为admin用户创建快速设置环境变量的脚本
	
	cat > ~/keystonerc_admin << EOF
	export OS_USERNAME=admin
	export OS_TENANT_NAME=admin
	export OS_PASSWORD=123456
	export OS_AUTH_URL=http://127.0.0.1:35357/v2.0/
	export PS1="[\u@\h \W(keystone_admin)]\$ "
	EOF

不再使用`admin_token`的验证方式，清除环境变量

	unset SERVICE_TOKEN
	unset SERVICE_ENDPOINT
	
用admin用户测试一下 (如果keystone用`-d`参数启动，还能看到debug信息)

	. ~/keystonerc_admin

	keystone user-list
	+----------------------------------+-------+---------+-------+
	|                id                |  name | enabled | email |
	+----------------------------------+-------+---------+-------+
	| 94d416d8ebf34d3e97e345afcc5a2283 | admin |   True  |       |
	+----------------------------------+-------+---------+-------+

### 创建用户账户

创建用户、角色、租户

	keystone user-create --name joe --pass 123123
	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|  email   |                                  |
	| enabled  |               True               |
	|    id    | 770def1aa63847bb8a5d31e1df2004d5 |
	|   name   |               joe                |
	| tenantId |                                  |
	+----------+----------------------------------+
	
	keystone role-create --name user
	+----------+----------------------------------+
	| Property |              Value               |
	+----------+----------------------------------+
	|    id    | 430f055d142649d1b02e685025b07a5b |
	|   name   |               user               |
	+----------+----------------------------------+

	keystone tenant-create --name trial
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |                                  |
	|   enabled   |               True               |
	|      id     | 249d66d789d643fabc544f6d6fc9ed9f |
	|     name    |              trial               |
	+-------------+----------------------------------+

将用户joe设置为user角色和租户trial的成员

	keystone user-role-add --user 770def1aa63847bb8a5d31e1df2004d5 \
	--role 430f055d142649d1b02e685025b07a5b \
	--tenant_id 249d66d789d643fabc544f6d6fc9ed9f
	
为用户joe创建环境变量快速设置脚本

	cat > ~/keystonerc_joe << EOF
	export OS_USERNAME=joe
	export OS_TENANT_NAME=trial
	export OS_PASSWORD=123123
	export OS_AUTH_URL=http://127.0.0.1:5000/v2.0/
	export PS1="[\u@\h \W(keystone_joe)]\$ "
	EOF
	
测试一下（`keystone user-list`只有管理员才有权限调用，应该会报错，但是`keystone token-get`可以返回信息）

	. ~/keystonerc_joe
	
	$ keystone user-list
	2013-09-08 14:10:14.312 10936 WARNING keystone.common.wsgi [-] You are not authorized to perform the requested action, admin_required.
	You are not authorized to perform the requested action, admin_required. (HTTP 403)
	
	keystone token-get

## 安装Swift

获取源码

	git clone git://github.com/openstack/swift.git

安装依赖

	cd swift
	pip install -r requirements.txt
	apt-get install memcached
	
注：pip安装依赖时可能报错`error: ffi.h: No such file or directory`，见`Troubleshooting`部分
	
安装swift到系统

	python setup.py install
	
### 创建Ring Files

Ring File包含存储设备的所有信息 （each ring will contain 2^12=4096 partitions）

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

	mkdir -p /etc/swift
	
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
	| d28d840d4be74a778578953a1a13324f | RegionOne |          http://127.0.0.1:5000/v2.0         |          http://127.0.0.1:5000/v2.0         |         http://127.0.0.1:35357/v2.0         | 714af530900840ef88106765f13f1921 |
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

创建和挂载loopback设备（注意：重启后挂载失效）

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
	
注：启动时可能会报错`NameError: name '_' is not defined`，解决方案见`Troubleshooting`部分
	
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

## 安装Glance

获取源码

	git clone git://github.com/openstack/glance.git
	
安装依赖

	cd glance
	pip install -r requirements.txt
	
安装Glance到系统

	python setup.py install
	
安装客户端

	pip install python-glanceclient
	
创建数据库

	mysql -u root -p
	create database glance;
	quit
	
复制配置文件

	mkdir -p /etc/glance
	cp etc/* /etc/glance/

修改`/etc/glance/glance-api.conf`
	
	[DEFAULT]
	# default_store = file
	default_store = swift

	swift_store_auth_address = http://127.0.0.1:35357/v2.0/
	swift_store_user = admin:admin
	swift_store_key = 123456
	swift_store_create_container_on_put = True
	
	[keystone_authtoken]
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456
	
	[paste_deploy]
	flavor = keystone

修改`/etc/glance/glance-registry.conf`

	[DEFAULT]
	#sql_connection = sqlite:///glance.sqlite
	sql_connection = mysql://root:111111@127.0.0.1/glance
	
	[keystone_authtoken]
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456
	
	[paste_deploy]
	flavor = keystone
	
初始化数据库

	mkdir -p /var/log/glance
	glance-manage db_sync

创建service和endpoint

	keystone service-create --name=glance --type=image --description="Glance Image Service"
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	| description |       Glance Image Service       |
	|      id     | c9f6e93cfd384a27bdac595be296ad4a |
	|     name    |              glance              |
	|     type    |              image               |
	+-------------+----------------------------------+	
	keystone endpoint-create --service_id c9f6e93cfd384a27bdac595be296ad4a \
	--publicurl http://localhost:9292/v1 \
	--adminurl http://localhost:9292/v1 \
	--internalurl http://localhost:9292/v1
	+-------------+----------------------------------+
	|   Property  |              Value               |
	+-------------+----------------------------------+
	|   adminurl  |     http://localhost:9292/v1     |
	|      id     | c052904b25ac44c785c9ecb4cd53507e |
	| internalurl |     http://localhost:9292/v1     |
	|  publicurl  |     http://localhost:9292/v1     |
	|    region   |            regionOne             |
	|  service_id | c9f6e93cfd384a27bdac595be296ad4a |
	+-------------+----------------------------------+

启动服务（运行每个服务都需要开一个终端，然后`vagrant ssh`登陆）

	glance-api --config-file /etc/glance/glance-api.conf
	glance-registry --config-file /etc/glance/glance-registry.conf

测试一下

	. ~/keystonerc_admin

	glance image-list
	+----+------+-------------+------------------+------+--------+
	| ID | Name | Disk Format | Container Format | Size | Status |
	+----+------+-------------+------------------+------+--------+
	+----+------+-------------+------------------+------+--------+

增加image

	head -c 1024 /dev/urandom > ~/image.file

	$ glance add name="test image" is_public=true disk_format=aki \
	container_format=aki < ~/image.file
	Added new image with ID: 32672784-6813-48c2-8c09-7ec99dd2c4a6
	
查看一下

	glance image-list
	+--------------------------------------+------------+-------------+------------------+------+--------+
	| ID                                   | Name       | Disk Format | Container Format | Size | Status |
	+--------------------------------------+------------+-------------+------------------+------+--------+
	| 1a736b75-3e49-4fed-97f1-f3257a75b3b8 | test image | aki         | aki              | 1024 | active |
	+--------------------------------------+------------+-------------+------------------+------+--------+

	$ swift list
	c1
	c2
	glance

	$ swift list glance
	1a736b75-3e49-4fed-97f1-f3257a75b3b8


##安装Cinder

获取源码

	git clone https://github.com/openstack/cinder.git
	git clone https://github.com/openstack/python-cinderclient.git

安装

	cd cinder
	pip install -r requirements.txt
	python setup.py  install
	
	cd ../python-cinderclient/
	pip install -r requirements.txt
	python setup.py  install
	
	apt-get install tgt open-iscsi rabbitmq-server
	
安装配置

	cp -af etc/cinder /etc
	cp /etc/cinder/cinder.conf.sample /etc/cinder/cinder.conf
	
修改`/etc/cinder/api-paste.ini`

	[filter:authtoken]
	paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
	service_protocol = http
	service_host = 127.0.0.1
	service_port = 5000
	auth_host = 127.0.0.1
	auth_port = 35357
	auth_protocol = http
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456

修改`/etc/cinder/cinder.conf`

	[DEFAULT]
	rootwrap_config=/etc/cinder/rootwrap.conf
	sql_connection=mysql://root:111111@127.0.0.1/cinder
	api_paste_config=/etc/cinder/api-paste.ini
	
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

	cinder-manage db sync
	
创建卷

	dd if=/dev/zero of=~/cinder-volumes bs=1 count=0 seek=2G
	losetup /dev/loop2 ~/cinder-volumes
	pvcreate /dev/loop2
	vgcreate cinder-volumes /dev/loop2
	
	$ pvscan
	PV /dev/sda5    VG precise64        lvm2 [79.76 GiB / 0    free]
	PV /dev/loop2   VG cinder-volumes   lvm2 [2.00 GiB / 2.00 GiB free]
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

##安装Nova

获取源码

	git clone git://github.com/openstack/nova.git
	git clone https://github.com/openstack/python-novaclient.git
	
安装源码

	cd nova
	pip install -r requirements.txt
	python setup.py  install
	cd ..
	
	cd python-novaclient
	pip install -r requirements.txt
	python setup.py  install
	cd ..
	
	
安装依赖（直接安装novnc会作为依赖安装比较早的novaclient包，可能导致错误）

	apt-get install python-libvirt
	
	apt-get download novnc
	dpkg --force-all -i novnc_*.deb

安装配置文件

	cp -af  nova/etc/nova /etc
	cp /etc/nova/nova.conf.sample /etc/nova/nova.conf

检查VT-X支持

	apt-get install cpu-checker
	kvm-ok
	
如果支持KVM，就会显示

	INFO: /dev/kvm exists
	KVM acceleration can be used

如果不支持

	INFO: Your CPU does not support KVM extensions
	KVM acceleration can NOT be used
	
我的虚机系统显示不支持，不能使用默认的KVM，所以就用QEMU测试

###QEMU

修改配置`/etc/nova/nova.conf`

	￼compute_driver=libvirt.LibvirtDriver
	libvirt_type=qemu

安装相关包

	apt-get install guestmount
	
###预配网络

将网卡设为`promiscuous mode`

	ip link set eth0 promisc on
	
修改网卡配置`/etc/network/interface`

	# The loopback network interface
	auto lo
	iface lo inet loopback

	# The primary network interface
	auto eth0
	iface eth0 inet dhcp

	# Bridge network interface for VM networks
	auto br100
	iface br100 inet static
	address 192.168.100.1
	netmask 255.255.255.0
	bridge_stp off
	bridge_fd 0

增加桥接设备（取名`br100`）

	apt-get install bridge-utils
	brctl addbr br100
	/etc/init.d/networking restart

检查一下

	$ ifconfig
	br100	Link encap:Ethernet  HWaddr 46:4c:e1:8e:79:73  
			inet addr:192.168.100.1  Bcast:192.168.100.255  Mask:255.255.255.0
			inet6 addr: fe80::444c:e1ff:fe8e:7973/64 Scope:Link
			UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
			RX packets:0 errors:0 dropped:0 overruns:0 frame:0
			TX packets:13 errors:0 dropped:0 overruns:0 carrier:0
			collisions:0 txqueuelen:0 
			RX bytes:0 (0.0 B)  TX bytes:1022 (1.0 KB)

	eth0	Link encap:Ethernet  HWaddr 08:00:27:88:0c:a6  
			inet addr:10.0.2.15  Bcast:10.0.2.255  Mask:255.255.255.0
			inet6 addr: fe80::a00:27ff:fe88:ca6/64 Scope:Link
			UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1500  Metric:1
			RX packets:125943 errors:0 dropped:0 overruns:0 frame:0
			TX packets:65909 errors:0 dropped:0 overruns:0 carrier:0
			collisions:0 txqueuelen:1000 
			RX bytes:101091933 (101.0 MB)  TX bytes:3925291 (3.9 MB)

	lo		Link encap:Local Loopback  
			inet addr:127.0.0.1  Mask:255.0.0.0
			inet6 addr: ::1/128 Scope:Host
			UP LOOPBACK RUNNING  MTU:16436  Metric:1
			RX packets:0 errors:0 dropped:0 overruns:0 frame:0
			TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
			collisions:0 txqueuelen:0 
			RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
	
###配置数据库

创建nova的数据库

	mysql -u root -p
	create database nova;
	quit

/*
###配置Cinder

修改`/etc/cinder/cinder.conf`

	rabbit_virtual_host = /

修改`/etc/nova/nova.conf`

	volume_api_class=nova.volume.cinder.API
*/

###配置Nova

`/etc/nova/nova.conf` 全部配置如下：

	# LOGS/STATE
	verbose=True
	logdir=/var/log/nova
	state_path=/var/lib/nova
	lock_path=/var/lock/nova
	rootwrap_config=/etc/nova/rootwrap.conf

	# SCHEDULER
	compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler

	# VOLUMES
	volume_api_class=nova.volume.cinder.API
	volume_driver=nova.volume.driver.ISCSIDriver
	volume_group=cinder-volumes
	volume_name_template=volume-%s
	iscsi_helper=tgtadm

	# DATABASE
	sql_connection=mysql://root:111111@127.0.0.1/nova

	# COMPUTE
	libvirt_type=qemu
	compute_driver=libvirt.LibvirtDriver
	instance_name_template=instance-%08x
	api_paste_config=/etc/nova/api-paste.ini

	# COMPUTE/APIS: if you have separate configs for separate services
	# this flag is required for both nova-api and nova-compute
	#allow_resize_to_same_host=True

	# APIS
	osapi_compute_extension=nova.api.openstack.compute.contrib.standard_extensions
	ec2_dmz_host=127.0.0.1
	s3_host=127.0.0.1
	enabled_apis=ec2,osapi_compute,metadata

	# RABBITMQ
	rabbit_host=localhost
	rabbit_port=5672
	rabbit_userid=guest
	rabbit_password=321321
	rabbit_virtual_host=/

	# GLANCE
	image_service=nova.image.glance.GlanceImageService
	glance_api_servers=127.0.0.1:9292

	# NETWORK
	network_manager=nova.network.manager.FlatDHCPManager
	force_dhcp_release=True
	dhcpbridge_flagfile=/etc/nova/nova.conf
	#firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
	# Change my_ip to match each host
	my_ip=127.0.0.1
	public_interface=eth0
	vlan_interface=eth0
	flat_network_bridge=br100
	flat_interface=eth0
	fixed_range=192.168.100.0/24

	# NOVNC CONSOLE
	novncproxy_base_url=http://127.0.0.1:6080/vnc_auto.html
	# Change vncserver_proxyclient_address and vncserver_listen to match each compute host
	vncserver_proxyclient_address=127.0.0.1
	vncserver_listen=127.0.0.1

	# AUTHENTICATION
	auth_strategy=keystone
	[keystone_authtoken]
	auth_host = 127.0.0.1
	auth_port = 35357
	auth_protocol = http
	admin_tenant_name = admin
	admin_user = admin
	admin_password = admin
	signing_dirname = /tmp/keystone-signing-nova

修改`/etc/nova/api-paste.ini`

	[filter:authtoken]
	admin_tenant_name = admin
	admin_user = admin
	admin_password = 123456

创建相应目录

	mkdir /var/log/nova

###初始化数据库

	nova-manage db sync
	
###启动服务

	# controller node
	nova-api &
	nova-conductor &
	nova-network &
	nova-scheduler &
	nova-novncproxy &

	# compute node
	nova-compute &
	#nova-network &

###创建虚机使用的网络

	nova-manage network create private --fixed_range_v4=192.168.100.0/24 --bridge_interface=br100
	
###验证一下

	$ nova-manage service list
	Binary           Host                                 Zone             Status     State Updated_At
	nova-conductor   precise64                            internal         enabled    :-)   2013-09-24 16:21:53
	nova-network     precise64                            internal         enabled    :-)   2013-09-24 16:21:53
	nova-scheduler   precise64                            internal         enabled    :-)   2013-09-24 16:21:56
	nova-compute     precise64                            nova             enabled    :-)   2013-09-24 16:21:51

	$ nova-manage version
	2013.2

`nova image-list`输出结果应该和`glance image-list`相同

	$ nova image-list
	+--------------------------------------+------------+--------+--------+
	| ID                                   | Name       | Status | Server |
	+--------------------------------------+------------+--------+--------+
	| 1a736b75-3e49-4fed-97f1-f3257a75b3b8 | test image | ACTIVE |        |
	+--------------------------------------+------------+--------+--------+
	
	$ glance image-list
	+--------------------------------------+------------+-------------+------------------+------+--------+
	| ID                                   | Name       | Disk Format | Container Format | Size | Status |
	+--------------------------------------+------------+-------------+------------------+------+--------+
	| 1a736b75-3e49-4fed-97f1-f3257a75b3b8 | test image | aki         | aki              | 1024 | active |
	+--------------------------------------+------------+-------------+------------------+------+--------+






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
    keystone-all &
    
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

