---
title: mysql与mongodb性能测试java程序更新
author: 阿辉
date: 2010-04-09T15:20:00+00:00
categories:
- Mongodb
tags:
- Mongodb
keywords:
- Mongodb
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
对之前写的java测试程序进行了更新，增加了一些参数和查询的测试：

http://farmerluo.googlecode.com/files/mongotest.rar

使用方法：
```bash
[root@web dist]# java -jar mongotest.jar
Usage:
mysql test:
java -jar mongotest.jar < mysql > < [select | update | insert] > < rows >  < host > < username > < password> 
mongo test:
java -jar mongotest.jar < mongo > < [select | update | insert] > < rows >  < host >
```
<!--more-->
然后用这个程序测试了1亿条数据的更新和插入：
```bash
[root@web dist]# ./run.sh
java -jar mongotest.jar mongo insert 100000000 192.168.20.24 dbtest
Mongo Exception:can’t say something
Mongo error code:-2
Mongo Exception:can’t say something
Mongo error code:-2
Mongo Exception:can’t say something
Mongo error code:-2

Total run time:13670 sec
Total rows:100000000
mongo insert Result:7315row/sec

java -jar mongotest.jar mongo update 100000000 192.168.20.24 dbtest
Total run time:7927 sec
Total rows:100000000
mongo update Result:12615row/sec

java -jar mongotest.jar mongo select 100000000 192.168.20.24 dbtest
Exception in thread “main” java.util.NoSuchElementException
at java.util.LinkedList$ListItr.next(LinkedList.java:698)
at com.mongodb.DBCursor._next(DBCursor.java:335)
at com.mongodb.DBCursor.next(DBCursor.java:413)
at mongotest.mongodb.mongodb_select(mongodb.java:111)
at mongotest.Main.main(Main.java:65)

因为网络问题，有些没有insert成功，只成功了99992390条记录。mongo server上会报这种错误：
Thu Apr  8 19:24:19 insert dbtest.test 159ms
Thu Apr  8 19:24:22 MessagingPort recv() got 1 bytes wanted 4, lft=3
Thu Apr  8 19:24:24 insert dbtest.test 104ms
Thu Apr  8 19:24:30 MessagingPort recv() got 2 bytes wanted 4, lft=2
Thu Apr  8 19:24:36 insert dbtest.test 136ms
Thu Apr  8 19:24:41 insert dbtest.test 130ms
Thu Apr  8 19:24:42 insert dbtest.test 327ms
Thu Apr  8 19:24:43 insert dbtest.test 130ms
Thu Apr  8 19:24:43 insert dbtest.test 270ms
Thu Apr  8 19:24:45 insert dbtest.test 138ms
Thu Apr  8 19:24:51 insert dbtest.test 332ms
Thu Apr  8 19:24:51 insert dbtest.test 140ms
Thu Apr  8 19:24:52 insert dbtest.test 256ms
Thu Apr  8 19:24:52 insert dbtest.test 101ms
Thu Apr  8 19:24:52 insert dbtest.test 246ms
Thu Apr  8 19:24:52 insert dbtest.test 105ms
Thu Apr  8 19:24:52 insert dbtest.test 151ms
Thu Apr  8 19:24:53 insert dbtest.test 396ms
Thu Apr  8 19:24:53 OpCounters::gotOp unknown op: 38752749
op: 38752749
Thu Apr  8 19:24:53   Assertion failure 0 db/../util/message.h 119
0x4dcf16 0x4e5ab4 0x5de633 0x68d5b2 0x3bd4c0b20b 0x3301606617 0x3300ad3c2d
/usr/bin/mongod(_ZN5mongo12sayDbContextEPKc+0xe6) [0x4dcf16]
/usr/bin/mongod(_ZN5mongo8assertedEPKcS1_j+0x154) [0x4e5ab4]
/usr/bin/mongod(_ZN5mongo16assembleResponseERNS_7MessageERNS_10DbResponseERK11sockaddr_in+0x223) [0x5de633]
/usr/bin/mongod(_ZN5mongo10connThreadEv+0x232) [0x68d5b2]
/usr/lib64/libboost_thread-mt.so.4(thread_proxy+0x6b) [0x3bd4c0b20b]
/lib64/libpthread.so.0 [0x3301606617]
/lib64/libc.so.6(clone+0x6d) [0x3300ad3c2d]
Thu Apr  8 19:24:53   AssertionException in connThread, closing client connection
Thu Apr  8 19:24:53 connection accepted from 192.168.20.21:23516 #13
Thu Apr  8 19:24:58 allocating new datafile /var/lib/mongo/dbtest.32, filling with zeroes…
Thu Apr  8 19:25:24 done allocating datafile /var/lib/mongo/dbtest.32, size: 2047MB, took 25.824 secs
Thu Apr  8 19:25:24 insert dbtest.test 25888ms
```


mongodb的查询性能要差一些：
```bash
[root@web dist]# java -jar mongotest.jar mongo select 1000000 192.168.20.24 dbname5
Total run time:290 sec
Total rows:1000000
mongo select Result:3448row/sec

[root@web dist]# java -jar mongotest.jar mongo select 10000000 192.168.20.24 dbname5
Total run time:2952 sec
Total rows:10000000
mongo select Result:3387row/sec
```

后面准备把这个程序改成多线程的测一测mongodb的并发性能…