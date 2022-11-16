---
title: innodb_log_file_size参数调整要慎之又慎！！！
author: 阿辉
date: 2011-06-30T16:09:00+00:00
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
 
如果innodb_log_file_size以前是256M,现在要调整到512M，那么更改配置后，你将无法启动mysql，这个参数调整特别是有数据时需要慎之又慎！！！发这篇个文章主要是为了提醒我自己，以前吃过这方面的亏，还好当时是测试服务器。

那万一碰到后怎么办呢？

先改回去试试，能成功启动的话再导出数据做备份。再：

<!--more-->

要STOP服务先，然后再删除原来的文件………
打开`/var/lib/mysql`
删除`ib_logfile0, ib_logfile1……..ib_logfilen`
再开启选项，成功启动。