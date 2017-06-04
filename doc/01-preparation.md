# Openstack Setup #

Detail of openstack setup on Ubuntu Server

## Preparation ##

### Enable BIOS dev name e.g eth# ###

`vi /etc/default/grub`

```
GRUB_CMDLINE_LINUX_DEFAULT="biosdevname=0"
GRUB_CMDLINE_LINUX="biosdevname=0"
```

`reboot`

### Set static network ###

`vi /etc/network/interfaces`

```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
	address 192.168.0.2
	netmask	255.255.255.0
	gateway 192.168.0.1
	dns-nameservers 202.156.1.16 218.186.2.16 218.186.2.6
```

`service networking restart` or `reboot`

### Enable IP Forwarding ###

`vi /etc/sysctl.conf`

```
net.ipv4.ip_forward = 1
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
```

### Install NTP ###

`apt-get install -y ntp`

`vi /etc/ntp.conf`

```
server 127.127.1.0
fudge 127.127.1.0 stratum 10
```
