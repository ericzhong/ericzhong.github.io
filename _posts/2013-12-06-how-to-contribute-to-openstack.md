---
layout: post
title: 如何提交代码给OpenStack
tags: openstack fuelweb
category: it
---

#代码提交流程
##注册帐号
1. 注册[Launchpad](https://launchpad.net/+login)帐号。
1. 加入[OpenStack基金会](https://www.openstack.org/join/)，根据具体情况（独立开发者或公司）选择相应的协议签署。
1. 打开[https://review.openstack.org/](https://review.openstack.org/)页面，点击`Sign In`，用Launchpad账户登录，然后进入`Settings`页面：
    * 上传`SSH Public Key`
    * 进入`Identities`页面，可增加更多开发邮箱，最终代码提交邮箱以`Profile`中看到的为准
    * 进入`Contact Information`页面，所有信息都要填写，否则提交时会报错

更多请参考：[How To Contribute](https://wiki.openstack.org/wiki/How_To_Contribute) （OpenStack首页 > Wiki）

##开发并提交代码
参考[Gerrit Workflow](https://wiki.openstack.org/wiki/GerritWorkflow)

基本流程如下（忽略前期配置）：

  * `git clone [repo]`
  * `git pull --ff-only origin master`
  * `git checkout -b TOPIC-BRANCH` (这个Branch要按照规则命名，`blueprint`或`bugfix`都有固定格式，如果是其它的类型就和你的reviewer协商一个分支名)
  * 修改代码
  * `git commit -a`
  * `git review`
  * 进入[https://review.openstack.org/](https://review.openstack.org/)页面查看提交

如果你的reviewer给你做了修改或注释，在页面上会以`Patch Set`的形式显示出来，你自己也可以再修改，但再次提交时要给`git commit`加上`--amend参数`，表示这是一次修改，而不是一个新的提交；

万一不小心忘了加`--amend`参数，且执行了`git review`传上去了，那么：

  1. 进入该新提交的页面，点击`Abandoned`按钮，把它废弃掉；
  1. 执行`git reset HEAD^ && git commit -a --amend && git review`，废弃本地最新的错误commit，再重新用`--amend`参数commit


#Fuel-Web开发
参考[Fuel Development Environment](http://docs.mirantis.com/fuel-dev/develop/env.html#setup-for-nailgun-unit-tests)，按照文档流程安装配置环境，直到`3.4.3`，用`Fake Mode`启动，然后通过浏览器打开。

如果启动有问题，可能是依赖没更新，参考[Running Nailgun in Fake Mode](https://github.com/stackforge/fuel-web/blob/master/docs/develop/env.rst#running-nailgun-in-fake-mode)，最好每次开发前都先更新一下依赖。

##Fuel-Web汉化
因汉化还没全部完成，官方把多语言切换隐藏了，去除[这里](https://github.com/stackforge/fuel-web/blob/master/nailgun/static/templates/common/footer.html#L6)的`hide`标记后即可在首页的右下角看到en/cn切换链接。

Fuel-Web使用[i18next](http://i18next.com/)库来翻译，资源文件是[translation.json](https://github.com/stackforge/fuel-web/blob/master/nailgun/static/i18n/translation.json)，具体转换代码格式参考官网代码即可。

有些地方会使用复数，比如`1 node`和`10 nodes`，i18next本身支持复数的处理，见[官方文档](http://i18next.com/pages/doc_features.html)的`simple plural`部分。

汉化Reviwer：Vitaly Kramskikh <vkramskikh@mirantis.com>，IRC:`freenode.net`,Channel:`#fuel`,NickName:`vk`

编码规范：

  * use words describing placement of strings like “button”, “title”, “summary”, “description”, “label” and place them at the end of key (like “apply_button”, “cluster_description”, etc.). One-word strings may look better without any of these suffixes.
  * do NOT use shortcuts (“bt” instead of “button”, “descr” instead of “description”, etc.)
  * nest keys if it makes sense, for example, if there is a few values for statuses, etc.
  * if some keys are used in a few places (for example, in utils), move them to “common.*” namespace

# Troubleshooting
## Permission denied (publickey)
执行`git review`时可能会报该错误，使用如下命令调试：

    ssh -vv -p 29418 [username]@review.openstack.org
	ssh-add
    ssh -vv -p 29418 [username]@review.openstack.org

参考：

* [Permission denied (publickey)](https://review.openstack.org/Documentation/error-permission-denied.html)
* [Generating SSH Keys](https://help.github.com/articles/generating-ssh-keys)
* [Agent admitted failure to sign](https://help.github.com/articles/error-agent-admitted-failure-to-sign)
