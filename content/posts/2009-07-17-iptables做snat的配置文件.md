---
title: iptables做SNAT的配置文件
author: 阿辉
date: 2009-07-17T11:17:00+00:00
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
---
```
Generated by iptables-save v1.2.11 on Thu Nov 20 16:30:43 2008
nat
:PREROUTING ACCEPT [0:0]
:POSTROUTING ACCEPT [1:92]
:OUTPUT ACCEPT [1:92]
-A POSTROUTING -s 10.11.12.0/255.255.255.0 -o eth0 -j SNAT –to-source 122.227.129.72
COMMIT
# Completed on Thu Nov 20 16:30:43 2008
# Generated by iptables-save v1.2.11 on Thu Nov 20 16:30:43 2008
filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:RH-Firewall-1-INPUT - [0:0]
-A INPUT -j RH-Firewall-1-INPUT
-A FORWARD -j RH-Firewall-1-INPUT
-A RH-Firewall-1-INPUT -i lo -j ACCEPT
-A RH-Firewall-1-INPUT -i eth1 -j ACCEPT
-A RH-Firewall-1-INPUT -p icmp –icmp-type any -j ACCEPT
-A RH-Firewall-1-INPUT -p 50 -j ACCEPT
-A RH-Firewall-1-INPUT -p 51 -j ACCEPT
-A RH-Firewall-1-INPUT -m state –state ESTABLISHED,RELATED -j ACCEPT
-A RH-Firewall-1-INPUT -s 10.11.12.0/24 -j ACCEPT
-A RH-Firewall-1-INPUT -m state –state NEW -m tcp -p tcp –dport 22 -j ACCEPT
-A RH-Firewall-1-INPUT -m state –state NEW -m tcp -p tcp –dport 80 -j ACCEPT
-A RH-Firewall-1-INPUT -m state –state NEW -p tcp -m tcp –dport 53 -j ACCEPT
-A RH-Firewall-1-INPUT -m state –state NEW -p udp -m udp –dport 53 -j ACCEPT
-A RH-Firewall-1-INPUT -j REJECT –reject-with icmp-host-prohibited
COMMIT
# Completed on Thu Nov 20 16:30:43 2008
```
<!--more-->