---
title: innodb_log_file_size设多大是合适的？
author: 阿辉
date: 2011-08-29T15:23:00+00:00
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
INNODB_LOG_FILE 过于小，会直接触发CHECKPOINT，导致频繁IO请求； 多大是合适的？
```
========
: (none) 16:13:13> pager grep sequence               
PAGER set to 'grep sequence'
: (none) 16:13:14>
: (none) 16:13:15>  SHOW engine innodb STATUSG SELECT sleep(60); SHOW engine innodb STATUSG               
Log sequence number 1450 485101299
1 row in set (0.09 sec)


1 row in set (1 min 0.01 sec)

Log sequence number 1450 505024667
1 row in set (0.00 sec)

: (none) 16:14:37> nopager
PAGER set to stdout
: (none) 16:14:43> select (505024667-485101299)/1024/1024;
+---------------------------------+
| (505024667-485101299)/1024/1024 |
+---------------------------------+
|                     19.00040436 |
+---------------------------------+
1 row in set (0.00 sec)
========
```
<!--more-->

Notice the log sequence number. That's the total number of bytes written to the transaction log.
我们在高峰期间采样可以得到，1分钟产生19M的日志； 我觉得这个INNODB LOG大小设成 19M*60=1140M 已经足够了；
60分钟是一个经验值， 你也可以适当调大，比如 500M，3个文件 ；这相对来说是安全的；
当然你也可以用以下命令来查看日志产生的大小：
show status like 'Innodb_os_log_written'; select sleep(60); show status like 'Innodb_os_log_written';