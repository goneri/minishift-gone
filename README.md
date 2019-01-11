# How to deploy a minishift on a libvirt server

Some days ago, I deployed a minishift instance on my private network to play
a bit. This is the notes that I took.


The topology.

- the libvirt node
    - Fedora 29
    - 192.168.1.4
- the router
    - Fedora 29
    - 192.168.1.1

## Prepare the network on the libvirt node

First, I don't want to use firewalld on this node, this is the first thing I turn off:

```shell
systemctl stop firewalld
systemctl disable firewalld
```


## Prepare the networks on the libvirt node

```shell
echo "<network>
  <name>docker-machines</name>
  <uuid>2d463cc2-74df-40a0-9f99-7c323bff6762</uuid>
  <bridge name='virbr1' stp='on' delay='0'/>
  <mac address='52:54:00:aa:f4:1c'/>
  <forward mode='route'/>
  <ip address='192.168.42.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.42.2' end='192.168.42.254'/>
    </dhcp>
  </ip>
</network>" > /tmp/docker-machines.xml
virsh net-create --file /tmp/docker-machines.xml
virsh net-start docker-machines
virsh net-autostart docker-machines

Now we can recreate the network without the NAT, we use the forward key
to enable the route mode. By default libvirt creates a default network.
We want to recreate it, but without the NAT enabled:

```shell
virsh net-undefine default
virsh net-destroy default
```shell
echo "<network>
  <name>default</name>
  <uuid>73f2401e-7315-4f1c-b895-80e5dfb67dd9</uuid>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:0e:6e:b4'/>
  <forward mode='route'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
    </dhcp>
  </ip>
</network>" > /tmp/default.xml
virsh net-create --file /tmp/default.xml
virsh net-start default
virsh net-autostart default
```

## A route to the minishift network 192.168.42.0/24

On the router:

```shell
[root@ipv4-router ~]# nmcli c modify maison +ipv4.routes '192.168.42.0/24 192.168.1.4'
[root@ipv4-router ~]# nmcli c up maison
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/3)
[root@ipv4-router ~]# ip r
default via 23.233.108.225 dev eth0 proto dhcp metric 101 
default via 192.168.1.4 dev eth1 proto static metric 102 
23.233.108.224/27 dev eth0 proto kernel scope link src 23.233.108.240 metric 101 
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.1 metric 102 
192.168.42.0/24 via 192.168.1.4 dev eth1 proto static metric 102
```

## Let's start minishift

Back on the libvirt node:

```shell
./minishift start
```
