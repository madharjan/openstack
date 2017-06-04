# Openstack Setup #

Detail of openstack setup on Ubuntu Server


## Install OpenStack - Swift ##

### Install Swift ###

`apt-get install swift swift-account swift-container swift-object xfsprogs swift-object-expirer`

### Register Swift in Keystone ###

`keystone user-create --name=swift --pass=Pa55w0rd`

`keystone user-role-add --user=swift --tenant=service --role=admin`

`keystone service-create --name=swift --type=object-store  --description="OpenStack Object Storage"`
  
`keystone endpoint-create  --service=swift
  --publicurl='http://ubuntu:9090/v1/AUTH_%(tenant_id)s'
  --internalurl='http://ubuntu:9090/v1/AUTH_%(tenant_id)s'
  --adminurl=http://ubuntu:9090`

`mkdir -p /etc/swift`

### Update Configuration ###

`vi /etc/swift/swift.conf`

```
[swift-hash]
# random unique string that can never change (DO NOT LOSE)
swift_hash_path_prefix = 8059fdb8-1547-11e4-904f-10c37b4f2454
swift_hash_path_suffix = 8bd9f0f8-1547-11e4-99c6-10c37b4f2454
```

### Create Storage Partition ###

`mkfs.xfs /dev/sda5`

`echo "/dev/sda5 /mnt/sda5 xfs noatime,nodiratime,nobarrier,logbufs=8 0 0" >> /etc/fstab`

`mkdir -p /mnt/sda5`

`mount /mnt/sda5`

`mkdir /mnt/sda5/1 /mnt/sda5/2 /mnt/sda5/3`

`chown -R swift:swift /mnt/sda5/*`

`mkdir /srv`

`for x in {1..3}; do ln -s /mnt/sda5/$x /srv/$x; done`

`mkdir -p /srv/1/node/sdb1 /srv/2/node/sdb2 /srv/3/node/sdb3`

`for x in {1..3}; do chown -R swift:swift /srv/$x/; done`

`mkdir /var/run/swift`

`chown -R swift:swift /var/run/swift`

`mkdir -p /var/cache/swift1 /var/cache/swift2 /var/cache/swift3`

`chown swift:swift /var/cache/swift*`


### Create Configuration ###

`vi /etc/rsyncd.conf`

```
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = 127.0.0.1

[account6012]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/account6012.lock

[account6022]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/account6022.lock

[account6032]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/account6032.lock

[container6011]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/container6011.lock

[container6021]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/container6021.lock

[container6031]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/container6031.lock

[object6010]
max connections = 25
path = /srv/1/node/
read only = false
lock file = /var/lock/object6010.lock

[object6020]
max connections = 25
path = /srv/2/node/
read only = false
lock file = /var/lock/object6020.lock

[object6030]
max connections = 25
path = /srv/3/node/
read only = false
lock file = /var/lock/object6030.lock
```
`vi /etc/default/rsync`

```
RSYNC_ENABLE=true
```

`service rsync restart`

Verify rsync config

`rsync rsync://pub@localhost/`

```
account6012
account6022
account6032
container6011
container6021
container6031
object6010
object6020
object6030
```

`vi /etc/rsyslog.d/10-swift.conf`

```
# Uncomment the following to have a log containing all logs together
#local1,local2,local3,local4,local5.*   /var/log/swift/all.log

# Uncomment the following to have hourly proxy logs for stats processing
#$template HourlyProxyLog,"/var/log/swift/hourly/%$YEAR%%$MONTH%%$DAY%%$HOUR%"
#local1.*;local1.!notice ?HourlyProxyLog

local1.*;local1.!notice /var/log/swift/proxy.log
local1.notice           /var/log/swift/proxy.error
local1.*                ~

local2.*;local2.!notice /var/log/swift/storage1.log
local2.notice           /var/log/swift/storage1.error
local2.*                ~

local3.*;local3.!notice /var/log/swift/storage2.log
local3.notice           /var/log/swift/storage2.error
local3.*                ~

local4.*;local4.!notice /var/log/swift/storage3.log
local4.notice           /var/log/swift/storage3.error
local4.*                ~

local5.*;local5.!notice /var/log/swift/storage4.log
local5.notice           /var/log/swift/storage4.error
local5.*                ~

local6.*;local6.!notice /var/log/swift/expirer.log
local6.notice           /var/log/swift/expirer.error
local6.*                ~
```

`vi rsyslog.conf`

```
$PrivDropToGroup adm
```

`mkdir -p /var/log/swift`

`chown -R syslog.adm /var/log/swift`

`chmod -R g+w /var/log/swift`

`service rsyslog restart`


### Install Swift Proxy ###

`apt-get install swift swift-proxy memcached python-keystoneclient python-swiftclient python-webob`


### Update Configutation ###

`vi /etc/swift/proxy-server.conf`

```
[DEFAULT]
bind_port = 8080
user = swift
workers = 1
log_facility = LOG_LOCAL1

[pipeline:main]
pipeline = healthcheck cache tempurl authtoken keystoneauth proxy-server

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true
account_autocreate = true

[filter:tempurl]
use = egg:swift#tempurl
methods = GET HEAD PUT
incoming_remove_headers = x-timestamp
incoming_allow_headers =
outgoing_remove_headers = x-object-meta-*
outgoing_allow_headers = x-object-meta-public-*

[filter:keystoneauth]
use = egg:swift#keystoneauth
operator_roles = Member,admin,swiftoperator

[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory

# Delaying the auth decision is required to support token-less
# usage for anonymous referrers ('.r:*').
delay_auth_decision = true

# auth_* settings refer to the Keystone server
auth_uri = http://ubuntu:5000
auth_protocol = http
auth_host = ubuntu
auth_port = 35357

# the service tenant and swift username and password created in Keystone
admin_tenant_name = service
admin_user = swift
admin_password = Pa55w0rd

[filter:cache]
use = egg:swift#memcache

[filter:catch_errors]
use = egg:swift#catch_errors

[filter:healthcheck]
use = egg:swift#healthcheck
```

`vi /etc/swift/object-expirer.conf`

```
[DEFAULT]
user = swift

log_name = object-expirer
log_facility = LOG_LOCAL6
log_level = INFO

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

# See object-expirer.conf-sample for options
```

`vi /etc/swift/account-server/1.conf`

```
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6012
workers = 1
user = swift
recon_cache_path = /var/cache/swift1

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
```

`vi /etc/swift/container-server/1.conf`

```
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6011
workers = 1
user = swift

recon_cache_path = /var/cache/swift1
allow_versions = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
```

`vi /etc/swift/container-reconciler.conf`

```
[DEFAULT]
# swift_dir = /etc/swift
user = swift
# You can specify default log routing here if you want:
# log_name = swift
# log_facility = LOG_LOCAL0
# log_level = INFO
# log_address = /dev/log
#
# comma separated list of functions to call to setup custom log handlers.
# functions get passed: conf, name, log_to_console, log_route, fmt, logger,
# adapted_logger
# log_custom_handlers =
#
# If set, log_udp_host will override log_address
# log_udp_host =
# log_udp_port = 514
#
# You can enable StatsD logging here:
# log_statsd_host = localhost
# log_statsd_port = 8125
# log_statsd_default_sample_rate = 1.0
# log_statsd_sample_rate_factor = 1.0
# log_statsd_metric_prefix =

[container-reconciler]
# reclaim_age = 604800
# interval = 300
# request_tries = 3

[pipeline:main]
pipeline = catch_errors proxy-logging cache proxy-server

[app:proxy-server]
use = egg:swift#proxy
# See proxy-server.conf-sample for options

[filter:cache]
use = egg:swift#memcache
# See proxy-server.conf-sample for options

[filter:proxy-logging]
use = egg:swift#proxy_logging

[filter:catch_errors]
use = egg:swift#catch_errors
# See proxy-server.conf-sample for options
```

`vi /etc/swift/object-server/1.conf`

```
[DEFAULT]
devices = /srv/1/node
mount_check = false
disable_fallocate = true
bind_port = 6010
workers = 1
user = swift

recon_cache_path = /var/cache/swift1

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
```

`vi /etc/swift/account-server/2.conf`

```
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6022
workers = 1
user = swift
recon_cache_path = /var/cache/swift2

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
```

`vi /etc/swift/container-server/2.conf`

```
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6021
workers = 1
user = swift

recon_cache_path = /var/cache/swift2
allow_versions = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
```

`vi /etc/swift/object-server/2.conf`

```
[DEFAULT]
devices = /srv/2/node
mount_check = false
disable_fallocate = true
bind_port = 6020
workers = 1
user = swift

recon_cache_path = /var/cache/swift2

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
```

`vi /etc/swift/account-server/3.conf`

```
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6032
workers = 1
user = swift
recon_cache_path = /var/cache/swift3

[pipeline:main]
pipeline = recon account-server

[app:account-server]
use = egg:swift#account

[filter:recon]
use = egg:swift#recon

[account-replicator]
vm_test_mode = yes

[account-auditor]

[account-reaper]
```

`vi /etc/swift/container-server/3.conf`

```
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6031
workers = 1
user = swift

recon_cache_path = /var/cache/swift3
allow_versions = true

[pipeline:main]
pipeline = recon container-server

[app:container-server]
use = egg:swift#container

[filter:recon]
use = egg:swift#recon

[container-replicator]
vm_test_mode = yes

[container-updater]

[container-auditor]

[container-sync]
```

`vi /etc/swift/object-server/3.conf`

```
[DEFAULT]
devices = /srv/3/node
mount_check = false
disable_fallocate = true
bind_port = 6030
workers = 1
user = swift

recon_cache_path = /var/cache/swift3

[pipeline:main]
pipeline = recon object-server

[app:object-server]
use = egg:swift#object

[filter:recon]
use = egg:swift#recon

[object-replicator]
vm_test_mode = yes

[object-updater]

[object-auditor]
```

`mv account-server.conf account-server.conf.orig`

`mv container-server.conf container-server.conf.orig`

`mv object-server.conf object-server.conf.orig`

### Create Swift Ring ###

**`cd /etc/swift`**

`swift-ring-builder account.builder create 10 3 1`

`swift-ring-builder account.builder add r1z1-127.0.0.1:6012/sdb1 1`

`swift-ring-builder account.builder add r1z2-127.0.0.1:6022/sdb2 1`

`swift-ring-builder account.builder add r1z3-127.0.0.1:6032/sdb3 1`

`swift-ring-builder account.builder rebalance`

`swift-ring-builder container.builder create 10 3 1`

`swift-ring-builder container.builder add r1z1-127.0.0.1:6011/sdb1 1`

`swift-ring-builder container.builder add r1z2-127.0.0.1:6021/sdb2 1`

`swift-ring-builder container.builder add r1z3-127.0.0.1:6031/sdb3 1`

`swift-ring-builder container.builder rebalance`

`swift-ring-builder object.builder create 10 3 1`

`swift-ring-builder object.builder add r1z1-127.0.0.1:6010/sdb1 1`

`swift-ring-builder object.builder add r1z2-127.0.0.1:6020/sdb2 1`

`swift-ring-builder object.builder add r1z3-127.0.0.1:6030/sdb3 1`

`swift-ring-builder object.builder rebalance`

**Verify Swift Rings**

`swift-ring-builder account.builder`

`swift-ring-builder container.builder`

`swift-ring-builder object.builder`

**Restart service**

`service swift-proxy restart`

### Restart Services ###

`swift-init all start`

### Verify Swift ###

`keystone role-create --name swiftoperator`

`keystone user-role-add --user <username> --tenant <tenant> --role swiftoperator`

`swift stat`

### Upload File ###

`swift upload <container> <file>`

### Create TempURL (signed) ###

`swift post -m "Temp-URL-Key:1234567890"`

`swift-temp-url GET 3600 /v1/AUTH_636b14af7558404db456bcb0a184ee9a/<container>/<file> 1234567890`

```
/v1/AUTH_636b14af7558404db456bcb0a184ee9a/<container>/<file>?temp_url_sig=0468ce84fe88ad48fb18ffb2e44bcb28b98f2c93&temp_url_expires=1406970541
```

The url `http://ubuntu:9090/v1/AUTH_636b14af7558404db456bcb0a184ee9a/<container>/<file>?temp_url_sig=0468ce84fe88ad48fb18ffb2e44bcb28b98f2c93&temp_url_expires=1406970541` will be **valid for 24 hrs**.




