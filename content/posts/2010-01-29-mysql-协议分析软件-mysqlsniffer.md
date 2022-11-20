---
title: mysql 协议分析软件 mysqlsniffer
author: 阿辉
date: 2010-01-29T14:51:00+00:00
categories:
- Mysql
tags:
- Mysql
keywords:
- Mysql
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
```bash
wget http://hackmysql.com/code/mysqlsniffer.tgz
tar -xvzf mysqlsniffer.tgz
yum install ibpcap-devel 
```

编译：  
`gcc -O2 -lpcap -o mysqlsniffer mysqlsniffer.c packet_handlers.c misc.c`

<!--more-->

用法：
```bash
[root@ha2 mysqlsniffer]# ./mysqlsniffer
mysqlsniffer v1.2 - Watch MySQL traffic on a TCP/IP network

Usage: mysqlsniffer [OPTIONS] INTERFACE

OPTIONS:
–port N        Listen for MySQL on port number N (default 3306)
–verbose       Show extra packet information
–tcp-ctrl      Show TCP control packets (SYN, FIN, RST, ACK)
–net-hdrs      Show major IP and TCP header values
–no-mysql-hdrs Do not show MySQL header (packet ID and length)
–state         Show state
–v40           MySQL server is version 4.0
–dump          Dump all packets in hex
–help          Print this

Original source code and more information at:
http://hackmysql.com/mysqlsniffer
```

可以过滤下：
```bash
./mysqlsniffer eth1 | grep ‘COM_QUERY’
```

网上有人直接tcpdump来捕捉，方法如下：
```bash
tcpdump -i eth1 -s 0 -l -w - dst port 3306 | strings | perl -e ‘
while(<>) { chomp; next if /^[^ ]+[ ]*/;
if(/^(SELECT|UPDATE|DELETE|INSERT|SET|COMMIT|ROLLBACK|CREATE|DROP|ALTER)/i) {
if (defined q) { print “qn”; }
q=_;
} else {
=~ s/^[ t]+//; q.=” _”;
}
}’
```