---
title: mysql slow 日志的一个bug
author: 阿辉
date: 2012-03-23T16:40:00+00:00
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
今天看mysql slow日志的时候发现有些查询的时间特别长，如下面：
```sql
# User@Host: flowershop2[flowershop2] @  [10.176.202.139]
# Query_time: 18446744073709.550781  Lock_time: 0.000027 Rows_sent: 0  Rows_examined: 1
SET timestamp=1332464662;
UPDATE f_users SET uCash = ‘9861’, uSellFlowerTime = ‘1332464662’ WHERE uId = ‘1829711071’;
# Time: 120322 21:06:22
# User@Host: flowershop2[flowershop2] @  [10.176.227.212]
# Query_time: 18446744073709.550781  Lock_time: 0.000022 Rows_sent: 0  Rows_examined: 1
SET timestamp=1332464782;
UPDATE f_user_actions SET uaSize = ’11’ WHERE uId = ‘1551376611’ AND uaId = ‘1332129864763’;
# Time: 120322 21:07:22
# User@Host: flowershop2[flowershop2] @  [10.176.202.139]
# Query_time: 18446744073709.550781  Lock_time: 0.000020 Rows_sent: 2  Rows_examined: 2
SET timestamp=1332464842;
```
<!--more-->

`Query_time: 18446744073709.550781`,显然这不是一个真实的查询时间。

网上搜了下，应该是mysql的一个bug:
http://bugs.mysql.com/bug.php?id=47813 

不过这个bug 09年就有人报了，好像现在还没有fix掉.

看到一篇老外的文章分析说可能是多cpu造成的，如一个查询开如由一个cpu,后来又转到第二个cpu去执行时，就可能会产生这个问题。

老外的文章：
http://www.mysqlperformanceblog.com/2009/05/17/what-time-18446744073709550000-means/