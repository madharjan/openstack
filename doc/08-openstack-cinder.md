# Openstack Setup #

Detail of openstack setup on Ubuntu Server


## Install OpenStack - Cinder ##

### Install Cinder ###

`apt-get install cinder-api cinder-scheduler cinder-volume`

### Create Database ###

`mysql -u root -p`

```
create database cinder;
grant all privileges on cinder.* to 'cinder'@'%' identified by 'Pa55w0rd';
flush privileges;
quit
```

### Update Configuration ###

`vi /etc/cinder/cinder.conf`

```
verbose = False

glance_host = ubuntu

rpc_backend = cinder.openstack.common.rpc.impl_kombu
rabbit_host = localhost
rabbit_port = 5672
rabbit_use_ssl = false
rabbit_userid = guest
rabbit_password = Pa55w0rd

[database]
connection = mysql://cinder:Pa55w0rd@localhost/cinder

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

`cinder-manage db sync`


### Register Cinder in Keystone ###

`keystone service-create --name=cinder --type=volume --description="OpenStack Block Storage"`

`keystone endpoint-create --service=cinder --publicurl=http://ubuntu:8776/v1/%\(tenant_id\)s --internalurl=http://ubuntu:8776/v1/%\(tenant_id\)s --adminurl=http://ubuntu:8776/v1/%\(tenant_id\)s`

`keystone service-create --name=cinderv2 --type=volumev2 --description="OpenStack Block Storage v2"`

`keystone endpoint-create --service=cinderv2 --publicurl=http://ubuntu:8776/v2/%\(tenant_id\)s --internalurl=http://ubuntu:8776/v2/%\(tenant_id\)s --adminurl=http://ubuntu:8776/v2/%\(tenant_id\)s`


### Restart Services ###

`service cinder-scheduler restart`

`service cinder-api restart`



### Configure Cinder Volumes ###

`pvcreate /dev/sda4`

`vgcreate cinder-volumes /dev/sda4`

`vi /etc/lvm/lvm.conf`

```
#filter = [ "a/.*/" ]
filter = [ "a/sda3/", "a/sda4/", "r/.*/" ]
 ```
 

### Restart Services ###

`service cinder-volume restart`

`service tgt restart`


### Test Cinder Configuration ###

`cinder create --display-name myvol 1`

`cinder list`

**Ubuntu Cloud Images**

`wget http://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img`

`qemu-img convert -O qcow2 trusty-server-cloudimg-amd64-disk1.img trusty-server-cloudimg-amd64-disk1.qcow2`

**View Image Details**

`qemu-img info trusty-server-cloudimg-amd64-disk1.qcow2`

**Upload Image**

`glance image-create --disk-format qcow2 --container-format bare --is-public True --name "ubuntu-14.04-64bit-cloudimg" --progress < trusty-server-cloudimg-amd64-disk1.qcow2`

**Using GUI** http://localhost/horizon

- Select Flavor - `m1.small`
- Select Boot Source - **Boot from image (create a new volume)** 
- Select Security Groups - `default`
- Select Key-Pairs - `cloud-key`
- Select Network - `private`
- Allocate Floating IP

**Note:** Root Disk size should be > 2.2GB

**Cloud-Config - (user-data)**

```
#cloud-config
password: Pa55w0rd
chpasswd: { expire: False }
ssh_pwauth: True
```

- Launch `Ubuntu` Instance with above `user-data`
- Associate Floating IP to the instance
- Test SSH to the Instance 
  
  `ssh -i cloud.key ubuntu@[floating_ip]`


