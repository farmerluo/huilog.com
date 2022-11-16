---
title: mysql 二进制日志轮询及innodb的写优化
author: 阿辉
date: 2011-04-22T14:25:00+00:00
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
innodb_buffer_pool_size
如果用Innodb，那么这是一个重要变量。相对于MyISAM来说，Innodb对于buffer size更敏感。MySIAM可能对于大数据量使用默认的key_buffer_size也还好，但Innodb在大数据量时用默认值就感觉在爬了。 Innodb的缓冲池会缓存数据和索引，所以不需要给系统的缓存留空间，如果只用Innodb，可以把这个值设为内存的70%-80%。和 key_buffer相同，如果数据量比较小也不怎么增加，那么不要把这个值设太高也可以提高内存的使用率。

innodb_additional_pool_size
这个的效果不是很明显，至少是当操作系统能合理分配内存时。但你可能仍需要设成20M或更多一点以看Innodb会分配多少内存做其他用途。

innodb_log_file_size
对于写很多尤其是大数据量时非常重要。要注意，大的文件提供更高的性能，但数据库恢复时会用更多的时间。我一般用64M-512M，具体取决于服务器的空间。

innodb_log_buffer_size
默认值对于多数中等写操作和事务短的运用都是可以的。如果经常做更新或者使用了很多blob数据，应该增大这个值。但太大了也是浪费内存，因为1秒钟总会 flush（这个词的中文怎么说呢？）一次，所以不需要设到超过1秒的需求。8M-16M一般应该够了。小的运用可以设更小一点。

innodb_flush_log_at_trx_commit  （这个很管用）
抱怨Innodb比MyISAM慢 100倍？那么你大概是忘了调整这个值。默认值1的意思是每一次事务提交或事务外的指令都需要把日志写入（flush）硬盘，这是很费时的。特别是使用电池供电缓存（Battery backed up cache）时。设成2对于很多运用，特别是从MyISAM表转过来的是可以的，它的意思是不写入硬盘而是写入系统缓存。日志仍然会每秒flush到硬盘，所以你一般不会丢失超过1-2秒的更新。设成0会更快一点，但安全方面比较差，即使MySQL挂了也可能会丢失事务的数据。而值2只会在整个操作系统挂了时才可能丢数据。

<!--more-->

上面是网上看的，我发现慢查询日志内有很多update和insert的查询，就把innodb_flush_log_at_trx_commit改成了2，效果很明显，改成0会更明显，但安全性比较差。做下面的操作启动mysqld就生效：
```
vim /etc/my.cnf
innodb_flush_log_at_trx_commit=2
```

也可以在mysqld运行时执行：
```
set GLOBAL innodb_flush_log_at_trx_commit = 2
```

下面是mysql手册上innodb_flush_log_at_trx_commit的解释：

如果innodb_flush_log_at_trx_commit设置为0，log buffer将每秒一次地写入log file中，并且log file的flush(刷到磁盘)操作同时进行；但是，这种模式下，在事务提交的时候，不会有任何动作。如果innodb_flush_log_at_trx_commit设置为1(默认值)，log buffer每次事务提交都会写入log file，并且，flush刷到磁盘中去。如果innodb_flush_log_at_trx_commit设置为2，log buffer在每次事务提交的时候都会写入log file，但是，flush(刷到磁盘)操作并不会同时进行。这种模式下，MySQL会每秒一次地去做flush(刷到磁盘)操作。注意：由于进程调度策略问题，这个“每秒一次的flush(刷到磁盘)操作”并不是保证100%的“每秒”。

默认值1是为了ACID (atomicity, consistency, isolation, durability)原子性，一致性，隔离性和持久化的考虑。如果你不把innodb_flush_log_at_trx_commit设置为1，你将获得更好的性能，但是，你在系统崩溃的情况，可能会丢失最多一秒钟的事务数据。当你把innodb_flush_log_at_trx_commit设置为0，mysqld进程的崩溃会导致上一秒钟所有事务数据的丢失。如果你把innodb_flush_log_at_trx_commit设置为2，只有在操作系统崩溃或者系统掉电的情况下，上一秒钟所有事务数据才可能丢失。InnoDB的crash recovery崩溃恢复机制并不受这个值的影响，不管这个值设置为多少，crash recovery崩溃恢复机制都会工作。
 
另外innodb_flush_method参数也值得关注，对写操作有影响：

innodb_flush_method： 设置InnoDB同步IO的方式：
   1) Default – 使用fsync（）。
   2) O_SYNC 以sync模式打开文件，通常比较慢。
   3) O_DIRECT，在Linux上使用Direct IO。可以显著提高速度，特别是在RAID系统上。避免额外的数据复制和double buffering（mysql buffering 和OS buffering）。

mysql二进制轮询的问题：
```bash
vi /etc/my.cnf
expire-logs-days = 7
```
这样mysql就自动会删除超过7天的二进制日志了。

手动删除可以用PURGE MASTER LOGS，下面也是网上看到的：
```sql
PURGE MASTER LOGS语法
PURGE {MASTER | BINARY} LOGS TO 'log_name'
PURGE {MASTER | BINARY} LOGS BEFORE 'date'
```
用于删除列于在指定的日志或日期之前的日志索引中的所有二进制日志。这些日志也会从记录在日志索引文件中的清单中被删除，这样被给定的日志成为第一个。
例如：
```sql
PURGE MASTER LOGS TO 'mysql-bin.010';
PURGE MASTER LOGS BEFORE '2003-04-02 22:46:26';
```
BEFORE变量的date自变量可以为'YYYY-MM-DD hh:mm:ss'格式。MASTER和BINARY是同义词。
如果您有一个活性的从属服务器，该服务器当前正在读取您正在试图删除的日志之一，则本语句不会起作用，而是会失败，并伴随一个错误。不过，如果从属服务器是休止的，并且您碰巧清理了其想要读取的日志之一，则从属服务器启动后不能复制。当从属服务器正在复制时，本语句可以安全运行。