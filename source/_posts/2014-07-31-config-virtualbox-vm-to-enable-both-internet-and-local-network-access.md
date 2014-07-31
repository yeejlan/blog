title: '配置Virtualbox虚拟机网卡，支持内外网访问'
date: 2014-07-31 09:30:00
tags: [virtualbox]
---

使用Virtualbox新建完成虚拟机后，打开设置->网络，第一块网卡选择"网络地址转换(NAT)"，用来共享上网；第二块网卡选择"桥接网卡",在网卡列表里面选择内网所连的那块网卡,这个虚拟网卡用来支持内网访问。

接下来启动虚拟机配置网卡:

CentOS系列:

配置第一块网卡，因为是NAT网络，直接使用dhcp服务自动配置

```config
#sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth0
TYPE=Ethernet
UUID=ccd8c9e7-f8a8-413d-a2eb-6c343f540ab1
#开机启用
ONBOOT=yes
NM_CONTROLLED=yes
#使用dhcp自动配置
BOOTPROTO=dhcp
HWADDR=08:00:27:D9:32:00
NAME="System eth0"
```

配置第二块网卡，桥接网络，需要手工配置

```config
#sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE=eth1
TYPE=Ethernet
UUID=741f9e69-c4f0-4321-a0ea-427611f53f8f
#开机启用
ONBOOT=yes
NM_CONTROLLED=yes
#手工配置
BOOTPROTO=static
HWADDR=08:00:27:78:C1:DF
#内网IP地址
IPADDR=192.168.1.101
#子网掩码，相当于255.255.255.0
PREFIX=24
#这里忽略了网关的配置
#GATEWAY=x.x.x.x
NAME="System eth1"
```

检查/etc/sysconfig/network，确保里面没有网关的配置

```config
NETWORKING=yes
HOSTNAME=vm3
#GATEWAY=x.x.x.x
```

sudo service network restart, ifconfig信息如下

```config
eth0   inet addr:10.0.2.15  
eth1   inet addr:192.168.1.101
```

route下确保缺省网关使用的是eth0(网络地址转换, aka NAT)的网关

```config
default         10.0.2.2        0.0.0.0         UG    0      0        0 eth0
```

这样配置后就既能通过eth0共享上网，又能通过eth1跟内网主机通信了。

Ubuntu系统配置如下:

```config
#sudo vi /etc/network/interfaces

auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
address 192.168.1.102
netmask 255.255.255.0
#gateway x.x.x.x
```

sudo ifup eth0 eth1，就可以了，如果上网有问题请检查下缺省网关，确保使用的是eth0所在网段的网关。