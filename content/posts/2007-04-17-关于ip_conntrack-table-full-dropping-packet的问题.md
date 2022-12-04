---
title: '关于ip_conntrack: table full, dropping packet的问题'
author: 阿辉
date: 2007-04-17T16:49:00+00:00
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
系统报：
```bash
Apr 17 16:37:26 www kernel: printk: 135 messages suppressed.
Apr 17 16:37:26 www kernel: ip_conntrack: table full, dropping packet.
Apr 17 16:37:26 www kernel: ip_conntrack: table full, dropping packet.
Apr 17 16:37:33 www kernel: printk: 139 messages suppressed.
Apr 17 16:37:33 www kernel: ip_conntrack: table full, dropping packet.
Apr 17 16:37:37 www kernel: printk: 60 messages suppressed.
Apr 17 16:37:37 www kernel: ip_conntrack: table full, dropping packet.
Apr 17 16:37:52 www kernel: printk: 19 messages suppressed.
Apr 17 16:37:52 www kernel: ip_conntrack: table full, dropping packet.
```

有丢包现象:
```bash
[root@www ~]# wc -l /proc/net/ip_conntrack
68463 /proc/net/ip_conntrack

[root@www ~]# cat /proc/sys/net/ipv4/netfilter/ip_conntrack_max
65536
```
<!--more-->
可调整以下参数：
```bash
echo 180 > /proc/sys/net/ipv4/netfilter/ip_conntrack_tcp_timeout_established
echo 262144 > /proc/sys/net/ipv4/netfilter/ip_conntrack_max
echo 120   > /proc/sys/net/ipv4/neigh/default/gc_stale_time
echo 1024 > /proc/sys/net/ipv4/neigh/default/gc_thresh1
echo 4096 > /proc/sys/net/ipv4/neigh/default/gc_thresh2
echo 8192 > /proc/sys/net/ipv4/neigh/default/gc_thresh3
```
或`vi /etc/sysctl.conf`
```bash
net.ipv4.ip_conntrack_max = 655360
net.ipv4.netfilter.ip_conntrack_tcp_timeout_established = 180
```
再`sysctl -p`

相关参数说明：

ARP 支持一个 sysctl 接口，可以用以配置全局参数或逐个网络接口地进行配制。 该 sysctl 可以通过`/proc/sys/net/ipv4/neigh//`文件或者使用 sysctl(2) 接口来访问。系统中每个接口都在`/proc/sys/net/ipv4/neigh/.`中有自己的目录。`default’目录中的设置用于所有新建的设备。 sysctl 相关的时间是以秒为单位，除非特别声明过．

anycast_delay
对 IPv6 相邻请求信息的回复的最大延迟时间； 目前还不支持 anycast。缺省值为1秒。

app_solicit
这是在使用多路广播探测(multicast probe)前， 经过网络连接送到用户间隙ARP端口监控程序的探测（probe） 最大数目(见 mcast_solicit )。 缺省值为0。

base_reachable_time
一旦发现相邻记录，至少在一段介于 base_reachable_time/2和3*base_reachable_time/2 之间的随机时间内，该记录是有效的。如果收到上层协议的肯定反馈， 那么记录的有效期将延长。 缺省值是30秒。

delay_first_probe_time
发现某个相邻层记录无效(stale)后，发出第一个探测要等待的时间。 缺省值是5秒。

gc_interval
收集相邻层记录的无用记录的垃圾收集程序的运行周期，缺省为30秒。

gc_stale_time
决定检查一次相邻层记录的有效性的周期。 当相邻层记录失效时，将在给它发送数据前，再解析一次。 缺省值是60秒。

gc_thresh1
存在于ARP高速缓存中的最少层数，如果少于这个数， 垃圾收集器将不会运行。缺省值是128。

gc_thresh2
保存在 ARP 高速缓存中的最多的记录软限制。 垃圾收集器在开始收集前，允许记录数超过这个数字 5 秒。 缺省值是 512。

gc_thresh3
保存在 ARP 高速缓存中的最多记录的硬限制， 一旦高速缓存中的数目高于此， 垃圾收集器将马上运行。缺省值是1024。

locktime
ARP 记录保存在高速缓存内的最短时间（jiffy数）， 以防止存在多个可能的映射(potential mapping)时， ARP 高速缓存系统的颠簸 (经常是由于网络的错误配置而引起)。 缺省值是 1 秒。

mcast_solicit
在把记录标记为不可抵达的之前， 用多路广播/广播（multicast/broadcast）方式解析地址的最大次数。 缺省值是3。

proxy_delay
当接收到有一个请求已知的代理 ARP 地址的 ARP 请求时， 在回应前可以延迟的 jiffy（时间单位，见BUG）数目。 这样，以防止网络风暴。缺省值是0.8秒。

proxy_qlen
能放入代理 ARP 地址队列(proxy-ARP addresses)的数据包最大数目。缺省值是64。

retrans_time
重发一个请求前的等待 jiffy（时间单位，见BUG）的数目。缺省值是1秒。

ucast_solicit
询问ARP端口监控程序前，试图发送单探测（unicast probe）的次数。 (见 app_solicit). 缺省值是3秒。

unres_qlen
每个没有被其它网络层解析的地址，在队列中可存放包的最大数目。缺省值是3.
