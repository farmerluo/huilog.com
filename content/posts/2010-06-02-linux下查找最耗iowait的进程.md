---
title: linux下查找最耗iowait的进程
author: 阿辉
date: 2010-06-02T14:02:00+00:00
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
抓哪个进程干坏事前要先停掉syslog`service syslog stop`

打开block dump:`echo 1 > /proc/sys/vm/block_dump`

统计：
```bash
dmesg | egrep “READ|WRITE|dirtied” | egrep -o ‘([a-zA-Z]*)’ | sort | uniq -c | sort -rn | head
1423 kjournald
1075 pdflush
209 indexer
3 cronolog
1 rnald
1 mysqld
```

<!--more-->

不要忘记在抓完之 后关掉block_dump和启动syslog:
```bash
echo 0 > /proc/sys/vm/block_dump
service syslog start
```

加个有人写了一个perl脚本来处理输出，能得到更直观的结果：

参考：
http://www.xaprb.com/blog/2009/08/23/how-to-find-per-process-io-statistics-on-linux/
```bash
wget http://aspersa.googlecode.com/svn/trunk/iodump
echo 1 > /proc/sys/vm/block_dump
while true; do sleep 1; dmesg -c; done | perl iodump

root@kanga:~# while true; do sleep 1; dmesg -c; done | perl iodump
^C# Caught SIGINT.
TASK PID TOTAL READ WRITE DIRTY DEVICES
firefox 4450 4538 251 4287 0 sda4, sda3
kjournald 2100 551 0 551 0 sda4
firefox 28452 185 185 0 0 sda4
kjournald 782 59 0 59 0 sda3
pdflush 31 30 0 30 0 sda4, sda3
syslogd 2485 2 0 2 0 sda3
firefox 28414 2 2 0 0 sda4, sda3
firefox 28413 1 1 0 0 sda4
firefox 28410 1 1 0 0 sda4
firefox 28307 1 1 0 0 sda4
firefox 28451 1 1 0 0 sda4
```
