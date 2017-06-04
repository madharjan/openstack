# Openstack Setup #

Detail of openstack setup on Ubuntu Server

### Install Neutron LBaas ###

`apt-get install neutron-lbaas-agent` 

### Update Configuration ###

`vi /etc/neutron/neutron.conf`

```
service_plugins = router,lbaas

service_provider = LOADBALANCER:Haproxy:neutron.services.loadbalancer.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default
```

`vi /etc/neutron/lbaas.ini`

```
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
device_driver = neutron.services.loadbalancer.drivers.haproxy.namespace_driver.HaproxyNSDriver
```

`vi /etc/openstack-dashboard/local_settings.py`

```
OPENSTACK_NEUTRON_NETWORK = {
    'enable_lb': True,
    'enable_firewall': False,
    'enable_quotas': True,
    'enable_vpn': False,
    # The profile_support option is used to detect if an external router can be
    # configured via the dashboard. When using specific plugins the
    # profile_support can be turned on if needed.
    'profile_support': None,
    #'profile_support': 'cisco',
}
```

### Restart Services ###

`service neutron-server restart`

`service neutron-lbaas-agent restart`

`service apache2 restart`

### Test Ubuntu Cloud Image (cloud-init, lbaas) ###

**Create 2 Instance of Ubuntu Cloud Image**

- Select Flavor - `m1.medium`
- Select Boot Source - Boot from image
- Select Security Groups - `default`
- Select Key-Pairs - `cloud-key`
- Select Network - `private`
- Set `user-data` [View File](openstack-user-data.txt)
- Allocate Floating IP

**Note:** Root Disk size should be > 2.2GB

**Using command line**

`nova flavor-list`

`nova net-list`

`nova boot --flavor 1 --key-name cloud-key --image ubuntu-14.04-32bit-cloudimg --security-groups default --nic net-id=d9f03796-981e-4d62-891a-196b7e409d43 --user-data=user-data.txt ubuntu-01`

Update line `<p>Welcome to Nginx Server: #2</p>` on `user-data` before launching #2.

`nova boot --flavor 1 --key-name cloud-key --image ubuntu-14.04-32bit-cloudimg --security-groups default --nic net-id=d9f03796-981e-4d62-891a-196b7e409d43 --user-data=user-data.txt ubuntu-02`


**Create LB Pool**

`neutron subnet-list`

`neutron lb-pool-create --lb-method ROUND_ROBIN --name LB-Web --protocol HTTP --subnet-id 5f66f921-49d7-4f9b-8e01-bb8fbc328336` 

`nova list` 

`neutron lb-member-create --address 10.70.0.7 --protocol-port 80 LB-Web`

`neutron lb-member-create --address 10.70.0.8 --protocol-port 80 LB-Web`

`neutron lb-healthmonitor-create --delay 3 --type HTTP --max-retries 3 --timeout 30`

`neutron lb-healthmonitor-associate 0c52e57e-097d-4035-9393-e986070ad9f0 LB-Web`


`neutron lb-vip-create --name VIP-Web --protocol-port 80 --protocol HTTP --subnet-id 5f66f921-49d7-4f9b-8e01-bb8fbc328336 LB-Web`

**Associate Floating IP to VIP using GUI**

**and test using script below**

`$while (true); do curl -sS http://192.168.0.24 | grep Welcome; sleep 2; done`


```
Welcome to Nginx Server: #1
Welcome to Nginx Server: #2
Welcome to Nginx Server: #1
Welcome to Nginx Server: #2
```
