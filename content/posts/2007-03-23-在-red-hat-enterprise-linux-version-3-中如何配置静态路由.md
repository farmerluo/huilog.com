---
title: 在 Red Hat Enterprise Linux version 3 中如何配置静态路由?
author: 阿辉
date: 2007-03-23T15:27:00+00:00
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
如果是在Red Hat Enterprise Linux 系统上第一次配置，添加静态路由需要创建一个新的文件。 文件名为`/etc/sysconfig/network-scripts/route-ethX`，其中X是你要给哪个网卡加静态路由的接口号码。 这个文件需要三个域： GATEWAY, NETMASK, 和 ADDRESS。 每个域后面必须有一个数字来表示这个域与那个路由相关。下面这个例子表示为eth0这个网络接口配置了两个静态路由。 
```bash
/etc/sysconfig/network-scripts/route-eth0  
GATEWAY0=10.10.0.1  
NETMASK0=255.0.0.0  
ADDRESS0=10.0.0.0 

GATEWAY1=10.2.0.1  
NETMASK1=255.255.0.0  
ADDRESS1=192.168.0.0 
```

<!--more-->
当完成上面的工作后需要重新启动系统或者重新启动网络： 

`service network restart`