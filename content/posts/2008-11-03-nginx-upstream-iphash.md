---
title: nginx –upstream iphash
author: 阿辉
date: 2008-11-03T11:06:00+00:00
categories:
- Nginx
tags:
- Nginx
keywords:
- Nginx
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
一般做负载均衡，都需要后端多台web服务器之间实现session共享，否则用户登录可能就有问题了。

今天看nginx文档时候，发现nginx可以根据客户端IP进行负载均衡，在upstream里设置ip_hash，就可以针对同一个C类地址段中的客户端选择同一个后端服务器，除非那个后端服务器宕了才会换一个。

原文如下：

The key for the hash is the class-C network address of the client. This method guarantees that the client request will always be forwarded to the same server. But if this server is considered inoperative, then the request of this client will be transferred to another server. This gives a high probability clients will always connect to the same server.

<!--more-->

也就是说我可以在两台服务器上跑两个论坛，但共享一个后台数据库，而不用去关心session共享的问题，前面启用ip_hash，正常情况下，客户端上网 获得IP，登录浏览发帖等都会被转到固定的后端服务器上，这样就不会出问题了。应该说来访IP分布越广，负载均衡就越平均。嘿嘿，如果都是同一个C来的用 户，那就没用了。

虚拟机上搭了个环境测试了一下：

装了三个nginx，分别在80，81，82端口上。

80端口上的nginx做负载均衡前端，配置到后面两个nginx：
```
upstream test{
            ip_hash;
            server 127.0.0.1:81 ;
            server 127.0.0.1:82 ;
        }
```
在81端口的nginx上写个简单的html，内容为1；在82端口的nginx上写个内容为2的html，两个文件同名。

在有ip_hash的时候，刷新页面http://192.168.1.33/index.html，始终显示为1，没有ip_hash的时候，则为轮流的1和2