---
title: 用netstat命令发现大量的TIME_WAIT现象：
author: 阿辉
date: 2007-04-17T23:41:00+00:00
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
用netstat命令发现大量的TIME_WAIT现象：
```
netstat -ae|grep 1521|grep root
……
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
TIME_WAIT   root
```
<!--more-->
检查`net.ipv4.tcp_tw`当前值，将当前的值更改为1分钟：
```bash
[root@aaa1 ~]# sysctl -a|grep net.ipv4.tcp_tw
net.ipv4.tcp_tw_reuse = 0
net.ipv4.tcp_tw_recycle = 0
[root@aaa1 ~]#

vi /etc/sysctl
#增加或修改net.ipv4.tcp_tw值：
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1

#使内核参数生效：
[root@aaa1 ~]# sysctl -p

[root@aaa1 ~]# sysctl -a|grep net.ipv4.tcp_tw
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_tw_recycle = 1
```
用netstat再观察正常


这里解决问题的关键是如何能够重复利用time_wait的值，我们可以设置时检查一下time和wait的值
```bash
#sysctl -a | grep time | grep wait
net.ipv4.netfilter.ip_conntrack_tcp_timeout_time_wait = 120
net.ipv4.netfilter.ip_conntrack_tcp_timeout_close_wait = 60
net.ipv4.netfilter.ip_conntrack_tcp_timeout_fin_wait = 120
```

打开tcp的连接复用:
```bash
sysctl -w net.ipv4.tcp_tw_reuse=1 #打开复用
sysctl -w net.ipv4.tcp_tw_recycle=10 #表示复用10次
```
或者:
```bash
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
echo 10 > /proc/sys/net/ipv4/tcp_tw_recycle
```
通过此方法，可以强制减少TCP的：time_wait连接，至于副作用，我还没发现:-)