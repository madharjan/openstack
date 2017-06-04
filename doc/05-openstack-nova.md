# Openstack Setup #

Detail of openstack setup on Ubuntu Server

## Install OpenStack - Nova ##

### Install Nova ###

`apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler python-novaclient nova-compute nova-console`


### Create Database ###

`mysql -u root -p`

```
create database nova;
grant all privileges on nova.* to 'nova'@'%' identified by 'Pa55w0rd';
flush privileges;
quit
```

### Register Nova in Keystone ###

`keystone user-create --name=nova --pass=Pa55w0rd`

`keystone user-role-add --user=nova --tenant=service --role=admin`

`keystone service-create --name=nova --type=compute --description="OpenStack Compute"`

`keystone endpoint-create --service=nova --publicurl=http://ubuntu:8774/v2/%\(tenant_id\)s --internalurl=http://ubuntu:8774/v2/%\(tenant_id\)s --adminurl=http://ubuntu:8774/v2/%\(tenant_id\)s`


### Update Configuration ###

`vi /etc/nova/nova.conf`

```
rpc_backend = rabbit
rabbit_host = localhost
rabbit_port = 5672
rabbit_use_ssl = false
rabbit_userid = guest
rabbit_password = Pa55w0rd

my_ip = 192.168.0.2
vncserver_listen = 192.168.0.2
vncserver_proxyclient_address = 192.168.0.2
novncproxy_base_url = http://ubuntu:6080/vnc_auto.html

glance_host = ubuntu
auth_strategy = keystone

network_api_class = nova.network.neutronv2.api.API
neutron_url = http://ubuntu:9696
neutron_auth_strategy = keystone
neutron_admin_tenant_name = service
neutron_admin_username = neutron
neutron_admin_password = Pa55w0rd
neutron_metadata_proxy_shared_secret = openstack
neutron_admin_auth_url = http://ubuntu:35357/v2.0
linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
security_group_api = neutron
service_neutron_metadata_proxy = True

vif_plugging_is_fatal: false
vif_plugging_timeout: 0

[database]
connection = mysql://nova:Pa55w0rd@localhost/nova

[keystone_authtoken]
auth_uri = http://ubuntu:5000
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = Pa55w0rd
```

### Initialize Database ###

`nova-manage db sync`


### Restart Services ###

`service nova-api restart`

`service nova-cert restart`

`service nova-consoleauth restart`

`service nova-scheduler restart`

`service nova-conductor restart`

`service nova-novncproxy restart`

`service nova-compute restart`

`service nova-console restart`

### Test Nova ###

`nova-manage service list`

```
Binary           Host                                 Zone             Status     State Updated_At
nova-cert        ubuntu                               internal         enabled    :-)   2014-07-19 07:18:16
nova-consoleauth ubuntu                               internal         enabled    :-)   2014-07-19 07:18:16
nova-scheduler   ubuntu                               internal         enabled    :-)   2014-07-19 07:18:16
nova-conductor   ubuntu                               internal         enabled    :-)   2014-07-19 07:18:16
nova-console     ubuntu                               internal         enabled    :-)   2014-07-19 07:18:17
nova-compute     ubuntu                               nova             enabled    :-)   2014-07-19 07:18:18
```
