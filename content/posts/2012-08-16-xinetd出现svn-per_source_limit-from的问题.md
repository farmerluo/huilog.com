---
title: xinetd出现svn per_source_limit from=的问题
author: 阿辉
date: 2012-08-16T07:25:12+00:00
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
今天机房网络出现问题，好了后发现svn一直连不上。查看日志发现有如下信息：
```
Aug 16 15:03:53 svnsh xinetd[19280]: EXIT: svn status=0 pid=29949 duration=1(sec)
Aug 16 15:04:27 svnsh xinetd[19280]: FAIL: svn per_source_limit from=112.64.23.235
```
网上查了一下，`per_source_limit from=`是xinetd的一个机制:

per_source

Takes an integer or "UNLIMITED" as an argument. This specifies the maximum instances of this service per source IP address. This can also be specified in the defaults section. 

instances

determines the number of servers that can be simultaneously active for a service (the default is no limit). The value of this attribute can be either a number or UNLIMITED which means that there is no limit. 

改成下面这样就好了，增加instances及instances:
<!--more-->
```
service svn
{
        disable = no
        per_source              = UNLIMITED
        instances               = UNLIMITED
        port                    = 3690
        socket_type             = stream
        protocol                = tcp
        wait                    = no
        user                    = root
        server                  = /usr/bin/svnserve
#        server_args             = -i -r /var/svn-repos
        server_args             = --log-file /var/log/svn.log -i -r /var/svn-repos
}
```

参考：
http://linux.die.net/man/5/xinetd.conf