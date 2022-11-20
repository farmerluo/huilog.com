---
title: mysql 慢查询分析脚本 mysqlsla
author: 阿辉
date: 2010-01-29T15:19:00+00:00
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
wget http://hackmysql.com/scripts/mysqlsla-2.03.tar.gz
tar -xvaf mysqlsla-2.03.tar.gz 
cd mysqlsla-2.03
```

安装：
```bash  
perl Makefile.PL  
make  
make install
```
<!--more-->

使用：  
```
mysqlsla -lt slow /var/log/mysql-slow-queries.log
```
作者建议配合mk-query-profiler 一起使用：
```bash
yum install perl-TermReadKey
wget http://maatkit.googlecode.com/files/maatkit-5427-1.noarch.rpm
rpm -ivh maatkit-5427-1.noarch.rpm
mysqlsla -lt slow /var/log/mysql-slow-queries.log -R print-unique -mf "db=jewelpuzzle" -sf "+SELECT" | mk-query-profiler -separate -database jewelpuzzle
```
貌似mk-query-profiler的参数用得有点不对，以后再研究下。