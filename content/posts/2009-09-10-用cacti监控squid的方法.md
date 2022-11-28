---
title: 用cacti监控squid的方法
author: 阿辉
date: 2009-09-10T14:31:00+00:00
categories:
- Cacti
tags:
- Cacti
keywords:
- Cacti
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
配置squid的snmp,安装时squid需加–enable-snmp参数。

`vi /usr/local/squid/etc/squid.conf`

加上下面几行：
```
snmp_port 3401
acl snmppublic snmp_community public
acl logger src 127.0.0.1/32
snmp_access allow snmppublic logger
snmp_access deny all
```

重启squid:
`service squid restart`

<!--more-->

用下面的命令测试：
`snmpwalk -v1 -c public 127.0.0.1:3401 .1.3.6.1.4.1.3495.1`

如果能看到类似下面的信息，说明成功了。
```
SNMPv2-SMI::enterprises.3495.1.1.1.0 = INTEGER: 16360
SNMPv2-SMI::enterprises.3495.1.1.2.0 = INTEGER: 21626872
…
```
因为squid的snmp用的是3401端口，而snmp默认端口是161，所以需要用net-snmpd做转发：

`vi /etc/snmp/snmpd.conf`

加上下面一行：
`proxy -v 1 -c public 127.0.0.1:3401 .1.3.6.1.4.1.3495.1`

重启net-snmpd:
`service snmpd restart`

再用下面的命令测试，如果能得到和上面一样的信息，就没问题了。
`snmpwalk -v1 -c public 127.0.0.1 .1.3.6.1.4.1.3495.1`

然后就是配置cacti了，导入 http://forums.cacti.net/about4142.html 的模板做图就行了。