# Openstack Setup #

Detail of openstack setup on Ubuntu Server

### Launch Instances - Cirros ###

***Create User***

`keystone user-create --name=demo --pass=Pa55w0rd --email=demo@example.com`

`keystone user-role-add --user=demo --tenant=stack --role=member`

***Using GUI*** [http://localhost/horizon](http://localhost/horizon)

  - Select Flavor - `m1.small`
  - Select Boot Source - Boot from image
  - Create Security Groups - `default`
  - Upload Key-Pairs - `cloud-key` [Private Key](cloud.key), [Public Key](cloud.pub)
  - Select Network - `private`
  - Allocate Floating IP
  - Launch `Cirros` Instance
  - Associate Floating IP to the instance
  
**Note**: Instance will not be accessible from External Network yet.


### Connecting to External Network ###

If **misconfigured, network access to the server will be broken**. Make sure you have access to server using console.

`vi /etc/network/interfaces`

```
#auto eth0
allow-br-ex eth0
iface eth0 inet manual
	up ifconfig $IFACE 0.0.0.0 up
	down ifconfig $IFACE down
	ovs_type OVSPort
	ovs_bridge br-ex

#auto br-ex
allow-ovs br-ex
iface br-ex inet static
	up ip link set $IFACE promisc on
	down ip link set $IFACE promisc off
	address 192.168.0.2
	netmask	255.255.255.0
	gateway 192.168.0.1
	dns-nameservers 202.156.1.16 218.186.2.16 218.186.2.6
	ovs_type OVSBridge
	ovs_ports eth0
```

`vi /etc/rc.local`

```
# By default this script does nothing.
ifup --allow=ovs br-ex
exit 0
```
### SSH to Instances ###

**From External Network**

`ssh cirros@[floating ip]`

password: cubswin:)