---
title: mac osx修改mac地址和管理路由表的方法
author: 阿辉
date: 2014-04-05T07:59:18+00:00
categories:
- MacOSX
tags:
- MacOSX
keywords:
- MacOSX
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
mac osx虽然是类unix的系统，默认也是bash shell，但是修改mac地址及管理路由表跟linux还是不一样。

修改mac地址,重启后失效
```
sudo ifconfig en0 lladdr d0:67:e5:2e:07:f1
```
修改路由表
```
sudo route delete 0.0.0.0  删除默认路由
sudo route add -net 0.0.0.0 192.168.1.1 默认使用192.168.1.1网关
sudo route add 10.0.1.0/24 10.200.22.254 其它网段指定网关
```
查看路由表（命令使用与linux不一样）
```
netstat -nr
```

<!--more-->