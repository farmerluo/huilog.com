---
title: 思科交换机的sysUptime时间问题
author: 阿辉
date: 2017-05-04T14:12:25+00:00
categories:
- Network
tags:
- Network
- Switcher
- cisco asa
- snmp
keywords:
- Network
- Switcher
- cisco asa
- snmp
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---
最近收到zabbix的监控报警，是一个交换机重启的误报，提示交换机重启了，但查看后发现实际并没有重启。

我们发现监控是否重启是监控交换同的引导时间，使用的是如下OID：
```
[root@vm-new-2 ~]# snmpwalk -v 2c -c public 10.99.88.25 1.3.6.1.2.1.1.3.0
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (2073092193) 239 days, 22:35:21.93
```

<!--more-->

登入设备上查看，发现并没有重启：
```
Cisco3750>show version
Cisco IOS Software, C3750E Software (C3750E-UNIVERSALK9-M), Version 12.2(53)SE2, RELEASE SOFTWARE (fc3)
Technical Support: http://www.cisco.com/techsupport
Copyright (c) 1986-2010 by Cisco Systems, Inc.
Compiled Wed 21-Apr-10 05:11 by prod_rel_team
Image text-base: 0x00003000, data-base: 0x02400000

ROM: Bootstrap program is C3750E boot loader
BOOTLDR: C3750E Boot Loader (C3750X-HBOOT-M) Version 12.2(53r)SE1, RELEASE SOFTWARE (fc1)

Cisco3750 uptime is 6 years, 5 weeks, 3 days, 8 hours, 15 minutes
System returned to ROM by power-on
System restarted at 13:36:39 GTM Tue Mar 29 2011
```
经过查阅相关资料，得知sysUpTime是由一个32-bit的counter来计数的，单位是1/100秒，所以最大时间为496天，过了496天就溢出，变成0，然后又重新计算时间.

做为变通的办法，可以使用另一个计数值来计算时间，那就是snmpEngineId (.1.3.6.1.6.3.10.2.1.3) ，其同样是32-bit的值，但它的单位是秒，所以可以存135年的运行时间，足够了。
```
[root@vm-new-2 ~]# snmpwalk -v 2c -c public 10.99.88.25 .1.3.6.1.6.3.10.2.1.3
SNMP-FRAMEWORK-MIB::snmpEngineTime.0 = INTEGER: 192528781 seconds
```