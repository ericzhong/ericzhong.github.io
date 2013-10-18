##在客户端建立公钥和私钥

在`~/.ssh`下生成公钥`id_rsa.pub`和私钥`id_rsa`

	ssh-keygen -t rsa		# 一路回车

##配置ssh服务器

修改`/etc/ssh/sshd_config`

	# 禁止root登录
	PermitRootLogin no

	# 防止客户端`~/.ssh`目录权限配置不对
	StrictModes no

	# 使用成对密钥鉴权
	RSAAuthentication yes
	PubkeyAuthentication yes
	AuthorizedKeysFile    .ssh/authorized_keys

	# 禁止密码登录
	PasswordAuthentication no

##将公钥放至ssh服务器

复制公钥到服务器

	scp ~/.ssh/id_rsa.pub <USER>@<IP>:~

登录ssh服务器,添加公钥

	cat id_rsa.pub >> ~/.ssh/authorized_keys

重启服务后配置生效

	/etc/init.d/sshd restart

##客户端访问

默认使用私钥`~/.ssh/id_rsa`

	ssh <USER>@<IP>	

也可以指定私钥

	ssh -i ~/.ssh/id_rsa <USER>@<IP>

