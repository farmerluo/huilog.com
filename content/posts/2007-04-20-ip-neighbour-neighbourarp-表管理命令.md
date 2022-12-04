---
title: ip neighbour–neighbour/arp 表管理命令
author: 阿辉
date: 2007-04-20T14:15:00+00:00
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
ip neighbour–neighbour/arp 表管理命令

缩写 neighbour、neighbor、neigh命令 add、change、replace、delete、fulsh、show(或者list)

`ip neighbour add`    －－－－－添加一个新的邻接条目

`ip neighbour change` －－－－－修改一个现有的条目

`ip neighbour replace` －－－－－替换一个已有的条目

<!--more-->

缩写：add、a；change、chg；replace、repl

例：在设备eth0上，为地址 10.0.0.3 添加一个 permanent ARP条目：
`# ip neigh add 10.0.0.3 lladdr 0:0:0:0:0:1 dev eth0 nud perm`

例：把状态改为 reachable
`# ip neigh chg 10.0.0.3 dev eth0 nud reachable`

`ip neighbour delete` －－－删除一个邻接条目

例：删除设备eth0上的一个ARP条目10.0.0.3
`# ip neigh del 10.0.0.3 dev eth0`

`ip neighbour show` －－ 显示网络邻居的信息. 
缩写：show、list、sh、ls

例：
```bash
# ip -s n ls 193.233.7.254
193.233.7.254. dev eth0 lladdr 00:00:0c:76:3f:85 ref 5 used 12/13/20 nud reachable
```

`ip neighbour flush` －－清除邻接条目. 缩写：flush、f

例： (-s 可以显示详细信息)
`# ip -s -s n f 193.233.7.254`