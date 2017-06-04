# Openstack Setup #

Detail of openstack setup on Ubuntu Server

### Ubuntu 14.04 Tahr / Openstack Ice House ###
With single interface **(eth0)**

## Preparation ##
[View Details](doc/01-preparation.md)

* ### Enable BIOS dev name e.g ethx ###

* ### Enable IP Forwarding ###

* ### Install NTP ###

## Install OpenStack - pre-requisite ##
[View Details](doc/02-prerequisite.md)

* ### Install Rabbit MQ ###

* ### Install MySQL ###

## Install OpenStack - packages ##

* ### Install Keystone ###
 [View Details](doc/03-openstack-keystone.md)

* ### Install Glance ###
 [View Details](doc/04-openstack-glance.md)

* ### Install Nova ###
 [View Details](doc/05-openstack-nova.md)

* ### Install Neutron ###
 [View Details](doc/06-openstack-neutron.md)
 
* ### Install Horizon ###

`apt-get install -y openstack-dashboard` 

`vi /etc/openstack-dashboard/local_settings.py`

```
OPENSTACK_ENDPOINT_TYPE = "internalURL"
```
 
* ### Launch Instance ###
 [View Details](doc/07-openstack-instance.md)
 

* ### Install Cinder ###
 [View Details](doc/08-openstack-cinder.md)
 
* ### Install Swift ###
  [View Details](doc/09-openstack-swift.md)
 
* ### Install LBaas ###
  [View Details](doc/10-openstack-lbaas.md)
 