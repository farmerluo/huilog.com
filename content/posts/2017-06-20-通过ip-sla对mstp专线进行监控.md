---
title: 通过ip sla+snmp方式对MSTP专线进行状态监控
author: 阿辉
date: 2017-06-20T08:46:01+00:00
categories:
- Network
tags:
- Network
- cisco router
- ip sla
- monitor
- snmp
keywords:
- Network
- cisco router
- ip sla
- monitor
- snmp
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---
MSTP专线由于是光纤通过光猫转换器转换成RJ45的电口到路由器，会导致一个问题，当链路上的光纤有中断时，接到路由器上的电口状态还是UP的，路由器对此无感知，这对我们的监控带来了困扰，因为通常对专线的监控都是监控接口的状态。

根据查询相关资料，监控的方式有以下几种：

*  1. 光猫转换器换成光电联动的光猫转换器，这种光猫转换器，光口down后，电口也会down,相应的路由器上的电口也就down了。据说有这种，我没有用过
*  2. 思科snmp协议有一个支持远程ping的OID，但是整个过程会很烦锁，当监控多条专线时，需要编程来处理。对这方面有兴趣的可参考：
http://www.cnblogs.com/cunshen/articles/163987.html 
以及 
http://www.cisco.com/c/en/us/support/docs/ip/simple-network-management-protocol-snmp/13383-21.html#examp
*  3. 使用路由器的ip sla与eem联动，原理是使用ip sla定时ping对方的接口地址，当发现对方的IP不通后，强制shutdown相应的专线接口。这种方式有一个问题，就是接口shutdown后，在运营商修复线路后，需要人工在设备上启动相应接口，不能自动总是不太好的，特别是节假日无人值班时。可参考
http://www.zhaocs.info/sla_eem_1.html

<!--more-->
以上几种监控方式都有缺点，当看到ip sla的监控方式后，我突然想到snmp+ip sla的监控方式，经测试确实可以，并可解决上面第3种方式需要人工介入的缺点。

配置方法：

# 1. 先在路由器上配置ip sla:

```
ip sla auto discovery
ip sla 1
 icmp-echo 196.10.12.242
 owner to_nanlingkeji_mplsvpn_196.10.12.242
 tag 1
 threshold 200  //超过200毫秒为超过阀值
 timeout 1000   //超时时间
 frequency 10   //每10秒执行一次
ip sla schedule 1 life forever start-time now
```

# 2. 查看ip sla状态：

```
3925-1#show ip sla statistics 1
IPSLAs Latest Operation Statistics

IPSLA operation id: 1
        Latest RTT: 4 milliseconds
Latest operation start time: 16:15:27 GTM Tue Jun 20 2017
Latest operation return code: OK   //状态
Number of successes: 318
Number of failures: 0
Operation time to live: Forever
```

# 3. 在监控服务器上，使用snmp查询ip sla状态：

```
[root@fft-vm-new-2 array]# snmpwalk -v 2c -c public 172.24.1.245 SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.2.1
SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.2.1 = STRING: "to_nanlingkeji_mplsvpn_196.10.12.242"
[root@fft-vm-new-2 array]# snmpwalk -v 2c -c public 172.24.1.245 1.3.6.1.4.1.9.9.42.1.2.10.1.2.1
SNMPv2-SMI::enterprises.9.9.42.1.2.10.1.2.1 = INTEGER: 1
[root@fft-vm-new-2 array]# snmpwalk -v 2c -c public 172.24.1.245 SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3
SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3.1 = STRING: "1"
SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3.902 = STRING: "902"
SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3.903 = STRING: "903"
SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3.905 = STRING: "905"
SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3.3999 = STRING: "3999"
```

注意：上面1.路由器配置内tag 1与上面snmp oid最后一位数字是对应的。
```
oid SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.2 为ip sla 的备注

oid  1.3.6.1.4.1.9.9.42.1.2.10.1.2为ip sla的状态
```
另外，可以使用oid SNMPv2-SMI::enterprises.9.9.42.1.2.1.1.3做为Discovery rule的key使用。


# 4. 有了上面几个OID，就可以在zabbix或其它监控软件上进行监控了。

下面是我导出的一个zabbix监控路由器的模板，有包括以上方式的监控：

http://www.huilog.com/wp-content/uploads/items/zbx\_export\_cisco\_router\_templates.xml


参考：https://networkengineering.stackexchange.com/questions/18078/what-are-the-snmp-mibs-for-cisco-ipsla-icmp-echo

http://snmp.cloudapps.cisco.com/Support/SNMP/do/BrowseOID.do?local=en&translate=Translate&objectInput=1.3.6.1.4.1.9.9.42.1.2.10.1.1