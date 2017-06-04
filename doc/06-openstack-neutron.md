# Openstack Setup #

Detail of openstack setup on Ubuntu Server

## Install OpenStack - Neutron ##

### Install Neutron ###

`apt-get install -y vlan bridge-utils`

`apt-get install -y neutron-server neutron-plugin-openvswitch neutron-plugin-openvswitch-agent neutron-common neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent openvswitch-switch`

### Create Database ###

`mysql -u root -p`

```
create database neutron;
grant all privileges on neutron.* to 'neutron'@'%' identified by 'Pa55w0rd';
flush privileges;
quit
```

### Register Neutron in Keystone ###

`keystone user-create --name=neutron --pass=Pa55w0rd`

`keystone service-create --name=neutron --type=network --description="OpenStack Networking"`

`keystone user-role-add --user=neutron --tenant=service --role=admin
keystone endpoint-create --service=neutron --publicurl http://ubuntu:9696 --adminurl http://ubuntu:9696  --internalurl http://ubuntu:9696`

`keystone tenant-get service | awk '$2~/^id/{print $4}'`


### Update Configuration ###

`vi /etc/neutron/neutron.conf`

```
core_plugin = ml2
#core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins = router
#service_plugins = neutron.services.l3_router.l3_router_plugin.L3RouterPlugin
allow_overlapping_ips = False
rpc_backend = neutron.openstack.common.rpc.impl_kombu
auth_strategy = keystone

rabbit_host = localhost
rabbit_password = Pa55w0rd
rabbit_port = 5672
rabbit_userid = guest

notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True

nova_url = http://ubuntu:8774/v2

nova_admin_username = nova
nova_admin_tenant_id = 618402f1765d4543bb99c89c2d674607
nova_admin_password = Pa55w0rd
nova_admin_auth_url = http://ubuntu:35357/v2.0

neutron_metadata_proxy_shared_secret = openstack
service_neutron_metadata_proxy = False

[keystone_authtoken]
auth_uri = http://ubuntu:5000
auth_host = 127.0.0.1
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = neutron
admin_password = Pa55w0rd
signing_dir = /var/cache/neutron

[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql://neutron:Pa55w0rd@localhost/neutron
```

**Configuration using - OVS: local, flat**

`vi /etc/neutron/plugins/ml2/ml2_conf.ini`


```
[ml2]
type_drivers = flat,vlan
tenant_network_types = vlan
mechanism_drivers = openvswitch

[ml2_type_flat]
flat_networks = physnet1,physnet2

[ml2_type_vlan]
network_vlan_ranges = physnet2:100:200

[securitygroup]
firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group=True

[ovs]
bridge_mappings = Intnet1:br-eth1
local_ip = 192.168.0.2

tenant_network_type = vlan
network_vlan_ranges = physnet2:100:200
```

`vi /etc/neutron/l3_agent.ini`

```
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
use_namespaces = True
l3_agent_manager = neutron.agent.l3_agent.L3NATAgentWithStateReport
external_network_bridge = br-ex
ovs_use_veth = True
root_helper = sudo /usr/local/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
```

`vi /etc/neutron/dhcp_agent.ini`

```
interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
use_namespaces = True
ovs_use_veth = True
root_helper = sudo /usr/local/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
dhcp_agent_manager = neutron.agent.dhcp_agent.DhcpAgentWithStateReport
```

`vi /etc/neutron/metadata_agent.ini`

```
admin_tenant_name = service
admin_user = neutron
admin_password = Pa55w0rd

metadata_proxy_shared_secret = openstack

auth_uri = http://192.168.0.3:5000
root_helper = sudo /usr/local/bin/neutron-rootwrap /etc/neutron/rootwrap.conf
nova_metadata_ip = 192.168.0.3
identity_uri = http://192.168.0.3:35357
```

### Configure vSwitch Bridges ###

`ovs-vsctl add-br br-eth1`

`ovs-vsctl add-br br-int`

`ovs-vsctl add-br br-ex`


### Restart Services ###

`service neutron-ovs-cleanup restart`

`service neutron-server restart`

`service neutron-plugin-openvswitch-agent restart`

`service neutron-l3-agent restart`

`service neutron-dhcp-agent restart`

`service neutron-metadata-agent restart`


### Create Public Network - OVS: flat,vlan ###

`tenant=$(keystone tenant-list | awk '/service/ {print $2}')`
`neutron net-create --tenant-id $tenant public --provider:network_type flat --provider:physical_network physnet1 --router:external=True`
`neutron subnet-create --ip_version 4 --gateway 192.168.0.1 --allocation-pool start=192.168.0.20,end=192.168.0.30 --enable_dhcp=False --name public-subnet public 192.168.0.0/24`

### Create Private Network ###

**Create Tenant - stack & demo**

`keystone tenant-create --name=stack`
`keystone tenant-create --name=demo`
 
**Create Private (tenant) Network**

Tenant `stack`

`tenant=$(keystone tenant-list | awk '/stack/ {print $2}')`
`neutron net-create --tenant-id $tenant private-stack --provider:network_type vlan --provider:physical_network physnet2 --provider:segmentation_id 101`
`neutron subnet-create --tenant-id $tenant --ip_version 4 --gateway 10.70.0.1 --name subnet-stack private-stack 10.70.0.0/24 --dns-nameservers list=true 202.156.1.16 218.186.2.16`

Tenant `demo`

`tenant=$(keystone tenant-list | awk '/demo/ {print $2}')`
`neutron net-create --tenant-id $tenant private-demo --provider:network_type vlan --provider:physical_network physnet2 --provider:segmentation_id 102`
`neutron subnet-create --tenant-id $tenant --ip_version 4 --gateway 10.70.1.1 --name subnet-demo private-demo 10.70.1.0/24 --dns-nameservers list=true 202.156.1.16 218.186.2.16`

### Create Router ###

Tenant `stack`

`neutron router-create --tenant-id $tenant router-stack`
`neutron router-gateway-set router-stack public`
`neutron router-interface-add router-stack subnet-stack`

Tenant `demo`

`neutron router-create --tenant-id $tenant router-demo`
`neutron router-gateway-set router-demo public`
`neutron router-interface-add router-demo subnet-demo`

**Configure Route**

`neutron port-list -c fixed_ips -c device_owner | grep router_gateway | awk -F '"' '{ print $8; }'`

`route add -net 10.70.0.0/24 gw $gateway1`
`route add -net 10.70.1.0/24 gw $gateway2`

### Delete LibVirt default network (optional) ###

`virsh net-destroy default`

`virsh net-autostart --disable default`

### Test Neutron ###

`neutron subnet-list`


