---
title: openvpn的安装配置及Replay-window backtrack occurred的问题
author: 阿辉
date: 2011-05-24T11:04:00+00:00
categories:
- OpenVPN
tags:
- OpenVPN
keywords:
- OpenVPN
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
为了更安全和方便的访问在机房服务器，我们在公司和机房间用openvpn建立一个vpn。

公司的内网网段是：192.168.20.0/24
机房的内网网段是：192.168.1.0/24
openvpn用的网段是：10.0.0.0/24
服务器和客户端都是Centos linux 5.x.

1) openvpn安装：
我是直接用yum从rpmforge库上安装的：
```bash
# yum install -y openvpn
```
客户端和服务器都是这么安装，如果你的机器上没有安装rpmforge,安装之前请先：
```bash
# rpm -ivh http://packages.sw.be/rpmforge-release/rpmforge-release-0.5.2-2.el5.rf.x86_64.rpm
```
<!--more-->

2) 服务端配置

a) 生成服务端的安全证书：
```bash
# cp -rf /usr/share/doc/openvpn-2.1.4/easy-rsa/2.0 /etc/openvpn/easy-rsa
# cd /etc/openvpn/easy-rsa/

# vim vars
# 按自己的需求更改下面几行：
export KEY_COUNTRY=CN
export KEY_PROVINCE=SHANGHAI
export KEY_CITY=SHANGHAI
export KEY_ORG=”SHOFFICE”
export KEY_EMAIL=”test@gmail.com”

# chmod 755 *

# source ./vars
# ./clean-all
# ./build-ca
# ./build-dh
# ./build-key-server server
```
然后把上面几个文件copy到/etc/openvpn目录中
```
keys/ca.crt
keys/Server.crt
keys/Server.key
keys/dh1024.pem
```

b) 接着把客户端密钥先生成了
```bash
# cd /etc/openvpn/easy-rsa/
# source ./vars
# ./build-key client1
# ./build-key client2
# ./build-key client3
...
```
上面生成client1,client2,client3三个客户端的密钥，key在keys目录下，把
```
ca.crt
clientx.crt
clientx.csr
clientx.key
```
这几个文件发给各自的客户端，x代表上面的1,2,3...

c) 服务器配置：
```
# vim /etc/openvpn/server.conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
dh dh1024.pem
mode server
tls-server
server 10.0.0.0 255.255.255.0
link-mtu 1300
push "route 192.168.1.0 255.255.255.0"
client-config-dir ccd
client-to-client
keepalive 5 30
tls-auth ta.key 0
comp-lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

分配客户端的IP段：
```bash
# mkdir ccd
# vim  ccd/client1
--ifconfig-push 10.0.0.2 10.0.0.1
# vim  ccd/client2
--ifconfig-push 10.0.0.5 10.0.0.6

# vim  ccd/client3
--ifconfig-push 10.0.0.9 10.0.0.10
```

服务器端的openvpn就配置好了，启动服务：
```bash
# service openvpn start
```

d) 服务器端iptables配置：
```bash
# vim /etc/sysconfig/iptables
*nat
:PREROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.0.0.0/24 -o eth1  -j SNAT --to-source 192.168.1.135
COMMIT
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:RH-Firewall-1-INPUT - [0:0]
-A INPUT -j RH-Firewall-1-INPUT
-A FORWARD -j RH-Firewall-1-INPUT
-A RH-Firewall-1-INPUT -i lo -j ACCEPT
-A RH-Firewall-1-INPUT -i eth1 -j ACCEPT
-A RH-Firewall-1-INPUT -i tap0 -j ACCEPT
-A RH-Firewall-1-INPUT -p icmp --icmp-type any -j ACCEPT
-A RH-Firewall-1-INPUT -p 50 -j ACCEPT
-A RH-Firewall-1-INPUT -p 51 -j ACCEPT
-A RH-Firewall-1-INPUT -p udp --dport 5353 -d 224.0.0.251 -j ACCEPT
-A RH-Firewall-1-INPUT -p udp -m udp --dport 631 -j ACCEPT
-A RH-Firewall-1-INPUT -p udp -m udp --dport 1194 -j ACCEPT
-A RH-Firewall-1-INPUT -p tcp -m tcp --dport 631 -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
-A RH-Firewall-1-INPUT -s 10.0.0.0/24 -j ACCEPT
-A RH-Firewall-1-INPUT -j REJECT --reject-with icmp-host-prohibited
COMMIT
```
有几个地方需要注意：
```
-A POSTROUTING -s 10.0.0.0/24 -o eth1  -j SNAT --to-source 192.168.1.135
```
这里是让客户端访问服务器的内网，如果是想让客户端通过这里共享上网，可以把192.168.1.135改成这台服务器上的公网IP就行了，当然，-o eth1也要改成相应的网卡。
```
-A RH-Firewall-1-INPUT -p udp -m udp --dport 1194 -j ACCEPT
```
需要开放openvpn的1194端口
```
-A RH-Firewall-1-INPUT -s 10.0.0.0/24 -j ACCEPT
```
需要让openvpn的网段通过

重启iptables:
```
# service iptables restart
```

还要更改内核参数，让其支持包转发：
```bash
# vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
```
让更改生效：
```bash
# sysctl -p
```

3) 客户端配置

a) 复制证书
       把服务器生成的客户端证书复制到客户端的/etc/openvpn目录内。

b) 配置openvpn
```bash
# vim /etc/openvpn.conf
client
dev tun
link-mtu 1300
proto udp
remote xxx.xxx.xxx.xxx 1194
resolv-retry infinite
nobind
ping 5
ping-restart 30
persist-local-ip
persist-remote-ip
ping-timer-rem
;persist-key
persist-tun
ca ca.crt
cert client1.crt
key client1.key
comp-lzo
status /var/log/openvpn-client-status.log
log /var/log/openvpn-client.log
```

启动openvpn：
```
# service openvpn start
```
应该已经能看到openvpn连上服务器了。

c) iptables配置：

```bash
# vim /etc/sysconfig/iptables
*nat
:PREROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 192.168.20.0/24 -d 192.168.1.0/24  -j SNAT --to-source 10.0.0.2
COMMIT
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:RH-Firewall-1-INPUT - [0:0]
-A INPUT -j RH-Firewall-1-INPUT
-A FORWARD -j RH-Firewall-1-INPUT
-A RH-Firewall-1-INPUT -i lo -j ACCEPT
-A RH-Firewall-1-INPUT -i eth0 -j ACCEPT
-A RH-Firewall-1-INPUT -i tun0 -j ACCEPT
-A RH-Firewall-1-INPUT -i tun1 -j ACCEPT
-A RH-Firewall-1-INPUT -p icmp --icmp-type any -j ACCEPT
-A RH-Firewall-1-INPUT -p 50 -j ACCEPT
-A RH-Firewall-1-INPUT -p 51 -j ACCEPT
-A RH-Firewall-1-INPUT -p udp --dport 5353 -d 224.0.0.251 -j ACCEPT
-A RH-Firewall-1-INPUT -p udp -m udp --dport 631 -j ACCEPT
-A RH-Firewall-1-INPUT -p udp -m udp --dport 1194 -j ACCEPT
-A RH-Firewall-1-INPUT -p tcp -m tcp --dport 631 -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 21 -j ACCEPT
-A RH-Firewall-1-INPUT -s 192.168.20.0/24 -j ACCEPT
-A RH-Firewall-1-INPUT -j REJECT --reject-with icmp-host-prohibited
COMMIT
```
重启iptables:
```bash
# service iptables restart
```

还要更改内核参数，让其支持包转发：
```bash
# vim /etc/sysctl.conf
net.ipv4.ip_forward = 1
```
让更改生效：
```bash
# sysctl -p
```
 
4) Replay-window backtrack occurred的问题

刚开始配置的时候，配置文件内并没有加link-mtu 1300这个参数。vpn也能正常用，但是用windows远程桌面的时候，有时会连不上，或者很慢。查看openvpn客户端的日志，发现有下面的类似报错：
```
Tue May 24 09:37:39 2011 Replay-window backtrack occurred [1]
Tue May 24 09:37:40 2011 Replay-window backtrack occurred [6]
```
解决方案有两个：

I.

google后发现可能是分片大小不当的问题，openvpn有下面几个参数是这一块的：
```
link-mtu 
mssfix
fragment
```
发现我这边只要加上link-mtu 1300就行了。

 

II.

改成tcp方式。
```
Error: "Replay-window backtrack occurred"
Sometimes network congestion and latency cause the UDP protocol, most commonly used with OpenVPN, to drop packets and even lose the connection. You will see a 'Replay window backtrack occurred' error in the log if this is occurring. One solution is to switch to the TCP protocol, assuming your server is configured to support a TCP connection.
```
参考下面的链接：

http://www.personalvpn.org/troubleshoot_openvpn.htm
