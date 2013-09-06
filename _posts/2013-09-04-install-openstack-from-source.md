#Troubleshooting
## Expecting an auth URL via either --os-auth-url or env[OS_AUTH_URL]
    export SERVICE_TOKEN=ADMIN
    export SERVICE_ENDPOINT=http://127.0.0.1:35357/v2.0/

    
## Unable to authorize user
    [catalog]
    driver = keystone.catalog.backends.templated.TemplatedCatalog
    template_file = default_catalog.templates

## [swift, pip install -r reuqirements.txt] ERROR： "c/_cffi_backend.c:14:17: fatal error: ffi.h: No such file or directory"
注释掉`requirements.txt`中的`xattr`

    pip install -r requirements.txt
    apt-get install python-xattr

## [swift-init all start] ERROR: "NameError: name '_' is not defined"
`/usr/local/lib/python2.7/dist-packages/keystone/exception.py` 增加：

    import gettext
    _ = gettext.gettext

## No module named swift_auth
`/etc/swift/proxy-server.conf` 修改：

    #keystone.middleware.swift_auth:filter_factory
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

## [glance add] Authorization Failed: <attribute 'message' of 'exceptions.BaseException' objects> (HTTP Unable to establish connection to http://127.0.0.1:35357/v2.0/tokens)
确认keystone服务是否启动

    ps -Af | grep keystone
    keystone-all &

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



