# Openstack Setup #

Detail of openstack setup on Ubuntu Server

## Install OpenStack - Keystone ##

### Install Keystone ###

`apt-get install -y keystone`

`mysql -u root -p`

```
create database keystone;
grant all privileges on keystone.* to 'keystone'@'%' identified by 'Pa55w0rd';
flush privileges;
quit
```

### Generate UUID ###

`apt-get install -y uuid`

`uuid`


### Update Configuration ###

`vi /etc/keystone/keystone.conf`

```
admin_token=8276a478-0f09-11e4-adc9-10c37b4f2454
#connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql://keystone:Pa55word@localhost/keystone
```

`rm /var/lib/keystone/keystone.db`


### Initialize Database ###

`service keystone restart`

`keystone-manage db_sync`


### Initialize Data (tenant, user, role, service) ###

`export OS_SERVICE_TOKEN=8276a478-0f09-11e4-adc9-10c37b4f2454`

`export OS_SERVICE_ENDPOINT=http://localhost:35357/v2.0`

`keystone tenant-create --name=admin --description="Admin Tenant"`

`keystone tenant-create --name=service --description="Service Tenant"`

`keystone user-create --name=admin --pass=Pa55w0rd`

`keystone role-create --name=admin`

`keystone user-role-add --user=admin --tenant=admin --role=admin`

`keystone service-create --name=keystone --type=identity --description="Keystone Identity Service"`

`keystone endpoint-create --service=keystone --publicurl=http://ubuntu:5000/v2.0 --internalurl=http://ubuntu:5000/v2.0 --adminurl=http://ubuntu:35357/v2.0`

### Create Credential Script ###

`vi ~/stack-admin.sh`

```
export OS_USERNAME=admin
export OS_PASSWORD=Pa55w0rd
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://localhost:35357/v2.0
```

`source ~/stack-admin.sh`

### Test Keystone ###

`keystone service-list`


