# Openstack Setup #

Detail of openstack setup on Ubuntu Server

## Install OpenStack - pre-requisite ##

### Install Rabbit MQ ###

`apt-get install -y rabbitmq-server`

`rabbitmqctl change_password guest Pa55w0rd`

### Install MySQL ###

`apt-get install -y mysql-server python-mysqldb`

`mysql_secure_installation`

`vi /etc/mysql/my.conf`

```
[mysqld]
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
```
