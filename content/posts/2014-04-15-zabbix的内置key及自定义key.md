---
title: zabbix的内置key及自定义key
author: 阿辉
date: 2014-04-15T07:22:03+00:00
categories:
- Zabbix
tags:
- Zabbix
keywords:
- Zabbix
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

以前使用nagios比较多，zabbix用得相对少一些。 发现zabbix对比nagios+cacti还是有些区别的：

* 1. zabbix的监控结果和数据全是存在数据库内，cacti是用一种rrd的文件DB。

* 2.zabbix有触发器，写监控脚本时只要把数据抓过来就行了，然后再去zabbix内配置触发器，做不同的报警。而nagios一般是直接写在脚本内了。

* 3.其它...以后再补充


zabbix的内置了很多key，如：

* 1、监控进程
```
/usr/local/zabbix/bin/zabbix_get -s 127.0.0.1 -k “net.tcp.service[http]”
```

* 2、监控端口
```
/usr/local/zabbix/bin/zabbix_get -s 127.0.0.1 -k “net.tcp.port[,80]”
```
结果：1存在，0不存在；

* 3、进程数量
```
/usr/local/zabbix/bin/zabbix_get -s 127.0.0.1 -k “proc.num[]”
/usr/local/zabbix/bin/zabbix_get -s 127.0.0.1 -k “proc.num[httpd]”
```

* 4、执行命令
```
/usr/local/zabbix/bin/zabbix_get -s 127.0.0.1 -k “system.run[curl -s  "http://127.0.0.1/php-fpm-uuzu-status"  | grep 'idle processes' | awk '{print $3;}']”
/usr/local/zabbix/bin/zabbix_get -s 127.0.0.1 -k “system.run[ps auxw | grep 'httpd' | grep -v 'grep' -c]”
```

<!--more-->

* 5、其他
```
vm.memory.size[available]
vfs.file.cksum[/etc/passwd]
system.cpu.switches
system.cpu.num
system.cpu.util[,user]
system.cpu.util[,nice]
system.cpu.util[,system]
system.cpu.util[,iowait]
system.cpu.util[,idle]
system.cpu.util[,interrupt]
system.cpu.util[,steal]
system.cpu.util[,softirq]
system.swap.size[,free]
system.swap.size[,pfree]
system.boottime
system.localtime
system.hostname
system.cpu.intr
kernel.maxfiles
kernel.maxproc
system.users.num
proc.num[]
proc.num[,,run]
system.cpu.load[percpu,avg1]
system.cpu.load[percpu,avg5]
system.cpu.load[percpu,avg15]
system.uname
system.uptime
vm.memory.size[total]
system.swap.size[,total]
net.tcp.service[ftp,,155]
net.tcp.service[http]
net.tcp.service.perf[http,,8080]
net.tcp.service[service,, ]
```

怎么自定义一些key呢？

要先启用自定义key，需要在客户端的配置文件zabbix_agentd.conf中启用UnsafeUserParameters=1参数，然后在配置文件的最下面来定义key，如：
```
UnsafeUserParameters=0 => UnsafeUserParameters=1并去掉前面的注释符
UserParameter=         => UserParameter=aaa.bbb[*], /usr/local/script/monitor.sh $1 $2 ...
```
说明：`aaa.bbb[*] ---zabbix服务器添加监控信息时需要用到的key值，`

格式：`aaa.bbb[*](例：system.file.size[*])`

`/usr/local/script/monitor.sh` ----监控脚本绝对路径

为了便于灵活监控，有时脚本需要传入参数，此参数可从zabbix服务器端传入，所有参数按顺序分别从$1-$9表示
注：   
* (1)若无需传入参数，则红色部分可省略
* (2)该自定义脚本可由zabbix服务器控制收集数据的频率（如：每30s运行一次），无需再添加计划任务
* (3)以上参数请根据实际情况填写，并注意去除参数前注释符(#)
* (4)注意在key值和后面的脚本之间有个逗号隔开