---
layout: default
title: Install OpenStack From Source
---


#Troubleshooting
## Expecting an auth URL via either --os-auth-url or env[OS_AUTH_URL]
```
export SERVICE_TOKEN=ADMIN
export SERVICE_ENDPOINT=http://127.0.0.1:35357/v2.0/
```
    
## Unable to authorize user
```
[catalog]
driver = keystone.catalog.backends.templated.TemplatedCatalog
template_file = default_catalog.templates
```

## [install swift] "pip install -r reuqirements.txt" ERROR： "c/_cffi_backend.c:14:17: fatal error: ffi.h: No such file or directory"

注释掉requirements.txt中的xattr
```
pip install -r requirements.txt
apt-get install python-xattr
```

## [swift-init all start] ERROR: "NameError: name '_' is not defined"

/usr/local/lib/python2.7/dist-packages/keystone/exception.py 增加：
```
import gettext
_ = gettext.gettext
```

## No module named swift_auth

/etc/swift/proxy-server.conf 修改：
```
#keystone.middleware.swift_auth:filter_factory
use = egg:swift#keystoneauth
```

## [swift list] Endpoint for object-store not found

/etc/keystone/default_catalog.templates 增加:
```
catalog.RegionOne.object-store.publicURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
catalog.RegionOne.object-store.adminURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
catalog.RegionOne.object-store.internalURL = http://localhost:8080/v1/AUTH_$(tenant_id)s
catalog.RegionOne.object-store.name = Object Store Service
```

## [swift-init all start] Unable to locate config for object-expirer

/etc/swift/object-server.conf 增加：
```
[object-expirer]
```

/etc/swift/object-expirer.conf
```
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
```

## [Log] No such file or directory: '/var/cache/swift/object.recon'

proxy-server.conf增加：
```
[filter:recon]
use = egg:swift#recon
recon_cache_path = /var/cache/swift
```
```
mkdir /var/cache/swift
chmod 777 /var/cache/swift
```


