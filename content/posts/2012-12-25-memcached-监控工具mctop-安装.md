---
title: memcached 监控工具mctop 安装
author: 阿辉
date: 2012-12-25T08:49:15+00:00
categories:
- Memcache
tags:
- Memcache
- Mctop
keywords:
- Memcache
- Mctop
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
mctop 与 top 相似，主要用于监视 Memcache 的流量，包括 key 的调用次数、对象存储大小、每秒的请求数、以及消耗的网络带宽等。

源代码：
https://github.com/etsy/mctop

1）安装：

```
[root@memcache2 mctop]# yum install libpcap-devel ruby-devel rubygems git
[root@memcache2 mctop]# gem install ruby-pcap -v '0.7.8'
[root@memcache2 mctop]# gem install bundle 
[root@memcache2 mctop]# gem install rake

[root@memcache2 mctop]# git clone git://github.com/etsy/mctop.git

[root@memcache2 mctop]# cd mctop/
[root@memcache2 mctop]# bundle install
[root@memcache2 mctop]# rake install
```
<!--more-->

2）运行
```
[root@memcache2 mctop]# mctop -h
Usage: mctop [options]
    -i, --interface=NIC              Network interface to sniff (required)
    -p, --port=PORT                  Network port to sniff on (default 11211)
    -d, --discard=THRESH             Discard keys with request/sec rate below THRESH
    -r, --refresh=MS                 Refresh the stats display every MS milliseconds
    -h, --help                       Show usage info

[root@memcache2 mctop]# mctop -i bond0 -p 11211
```

大致是这个样子： 

![/wp-content/uploads/2012/12/mctop.jpg](/wp-content/uploads/2012/12/mctop.jpg)
