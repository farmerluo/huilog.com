---
title: centos linux 7下使用nmcli绑定网卡
author: 阿辉
date: 2017-05-03T13:18:36+00:00
categories:
- Linux
tags:
- Linux
keywords:
- Linux
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false

---
在 CentOS 7.x 中可以用 nmcli（Network Manager Command Line Interface：网络管理命令行接口）进行网卡绑定。

多网卡的7种bond模式原理请参考： 
http://support.huawei.com/huaweiconnect/enterprise/thread-282727.html

官方文档：
https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/sec-Network_Bonding_Using_the_NetworkManager_Command_Line_Tool_nmcli.html

<!--more-->

 

nmcli需要NetworkManager服务，先启动：
```
# systemctl start NetworkManager
```
设备开机启动：
```
# systemctl enable NetworkManager
```

```
# systemctl status NetworkManager
 NetworkManager.service - Network Manager Loaded: loaded (/lib/systemd/system/NetworkManager.service; enabled) Active: active (running) since Fri, 08 Mar 2013 12:50:04 +0100; 3 days ago
```

增加设备：
```
[root@localhost ~]# nmcli con add type bond con-name bond0 ifname bond0 mode balance-rr
Connection 'bond0' (cfabe2d0-21e9-47e9-9286-c182659b8331) successfully added.
```

查看结果：
```
[root@localhost ~]# nmcli con show
NAME   UUID                                  TYPE            DEVICE
em3    6bcabcdd-656c-4db5-b5fe-6a62716e68f7  802-3-ethernet  --     
em2    e07ae4a6-19fd-4782-9dd9-87090ae00903  802-3-ethernet  --     
em4    e6cf4c05-f29a-4d3b-a550-b04878747362  802-3-ethernet  --     
bond0  cfabe2d0-21e9-47e9-9286-c182659b8331  bond            bond0  
em1    54bd8900-2e3b-4c69-a1cd-91366e88e0d3  802-3-ethernet  em1    
[root@localhost ~]#
```

增加网卡：
```
[root@localhost ~]# nmcli con add type bond-slave ifname em3 master bond0
Connection 'bond-slave-em3' (92cb4f6e-74ee-48a9-8e56-9f83716e11cc) successfully added.
[root@localhost ~]# nmcli con add type bond-slave ifname em4 master bond0
Connection 'bond-slave-em4' (3c1da71d-6608-41ff-82d4-2e2fd7582136) successfully added.
```

查看结果
```
[root@localhost ~]# nmcli con show
NAME            UUID                                  TYPE            DEVICE
em3             6bcabcdd-656c-4db5-b5fe-6a62716e68f7  802-3-ethernet  --     
em2             e07ae4a6-19fd-4782-9dd9-87090ae00903  802-3-ethernet  --     
em4             e6cf4c05-f29a-4d3b-a550-b04878747362  802-3-ethernet  --     
bond-slave-em4  3c1da71d-6608-41ff-82d4-2e2fd7582136  802-3-ethernet  em4    
bond-slave-em3  92cb4f6e-74ee-48a9-8e56-9f83716e11cc  802-3-ethernet  em3    
bond0           cfabe2d0-21e9-47e9-9286-c182659b8331  bond            bond0  
em1             54bd8900-2e3b-4c69-a1cd-91366e88e0d3  802-3-ethernet  em1    
[root@localhost ~]#
```

激活网卡：
```
[root@localhost ~]# nmcli con up bond-slave-em3
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/10)
[root@localhost ~]# nmcli con up bond-slave-em4
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/11)
```

配置IP地址等信息：
```
[root@localhost ~]# nmcli con mod bond0 ipv4.addresses "192.168.100.2/24"
[root@localhost ~]# nmcli con mod bond0 ipv4.method manual
```

启动网卡：
```
[root@localhost ~]# nmcli con down bond0
Connection 'bond0' successfully deactivated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/32)
[root@localhost ~]# nmcli con up bond0  
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/33)
```

查看结果：
```
[root@localhost ~]# ifconfig
bond0: flags=5187<UP,BROADCAST,RUNNING,MASTER,MULTICAST>  mtu 1500
        inet 192.168.100.2  netmask 255.255.255.0  broadcast 192.168.100.255
        ether 36:38:d7:a2:89:52  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

em1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.21.21.51  netmask 255.255.255.0  broadcast 172.21.21.255
        inet6 fe80::92b1:1cff:fe47:2d15  prefixlen 64  scopeid 0x20<link>
        ether 90:b1:1c:47:2d:15  txqueuelen 1000  (Ethernet)
        RX packets 17026  bytes 1250498 (1.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1091  bytes 152043 (148.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 41  

em2: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        ether 90:b1:1c:47:2d:16  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 46  

em3: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 90:b1:1c:47:2d:17  txqueuelen 1000  (Ethernet)
        RX packets 9  bytes 2551 (2.4 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 47  

em4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        ether 90:b1:1c:47:2d:18  txqueuelen 1000  (Ethernet)
        RX packets 4  bytes 492 (492.0 B)
        RX errors 0  dropped 48  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 48  

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 0  (Local Loopback)
        RX packets 1  bytes 99 (99.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1  bytes 99 (99.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[root@localhost ~]# cat /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE=bond0
TYPE=Bond
BONDING_MASTER=yes
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=bond0
UUID=cfabe2d0-21e9-47e9-9286-c182659b8331
ONBOOT=yes
BONDING_OPTS=mode=balance-rr
IPADDR=192.168.100.2
PREFIX=24
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
```

帮助：
```
[root@localhost ~]# nmcli con add help
Usage: nmcli connection add { ARGUMENTS | help }

ARGUMENTS := COMMON_OPTIONS TYPE_SPECIFIC_OPTIONS IP_OPTIONS

  COMMON_OPTIONS:
                  type <type>
                  ifname <interface name> | "*"
                  [con-name <connection name>]
                  [autoconnect yes|no]

                  [save yes|no]

  TYPE_SPECIFIC_OPTIONS:
    ethernet:     [mac <MAC address>]
                  [cloned-mac <cloned MAC address>]
                  [mtu <MTU>]

    wifi:         ssid <SSID>
                  [mac <MAC address>]
                  [cloned-mac <cloned MAC address>]
                  [mtu <MTU>]
                  [mode infrastructure|ap|adhoc]

    wimax:        [mac <MAC address>]
                  [nsp <NSP>]

    pppoe:        username <PPPoE username>
                  [password <PPPoE password>]
                  [service <PPPoE service name>]
                  [mtu <MTU>]
                  [mac <MAC address>]

    gsm:          apn <APN>
                  [user <username>]
                  [password <password>]

    cdma:         [user <username>]
                  [password <password>]

    infiniband:   [mac <MAC address>]
                  [mtu <MTU>]
                  [transport-mode datagram | connected]
                  [parent <ifname>]
                  [p-key <IPoIB P_Key>]

    bluetooth:    [addr <bluetooth address>]
                  [bt-type panu|dun-gsm|dun-cdma]

    vlan:         dev <parent device (connection  UUID, ifname, or MAC)>
                  id <VLAN ID>
                  [flags <VLAN flags>]
                  [ingress <ingress priority mapping>]
                  [egress <egress priority mapping>]
                  [mtu <MTU>]

    bond:         [mode balance-rr (0) | active-backup (1) | balance-xor (2) | broadcast (3) |
                        802.3ad    (4) | balance-tlb   (5) | balance-alb (6)]
                  [primary <ifname>]
                  [miimon <num>]
                  [downdelay <num>]
                  [updelay <num>]
                  [arp-interval <num>]
                  [arp-ip-target <num>]
                  [lacp-rate slow (0) | fast (1)]

    bond-slave:   master <master (ifname, or connection UUID or name)>

    team:         [config <file>|<raw JSON data>]

    team-slave:   master <master (ifname, or connection UUID or name)>
                  [config <file>|<raw JSON data>]

    bridge:       [stp yes|no]
                  [priority <num>]
                  [forward-delay <2-30>]
                  [hello-time <1-10>]
                  [max-age <6-40>]
                  [ageing-time <0-1000000>]
                  [mac <MAC address>]

    bridge-slave: master <master (ifname, or connection UUID or name)>
                  [priority <0-63>]
                  [path-cost <1-65535>]
                  [hairpin yes|no]

    vpn:          vpn-type vpnc|openvpn|pptp|openconnect|openswan|libreswan|ssh|l2tp|iodine|...
                  [user <username>]

    olpc-mesh:    ssid <SSID>
                  [channel <1-13>]
                  [dhcp-anycast <MAC address>]

    adsl:         username <username>
                  protocol pppoa|pppoe|ipoatm
                  [password <password>]
                  [encapsulation vcmux|llc]

  IP_OPTIONS:
                  [ip4 <IPv4 address>] [gw4 <IPv4 gateway>]
                  [ip6 <IPv6 address>] [gw6 <IPv6 gateway>]</pre>
```