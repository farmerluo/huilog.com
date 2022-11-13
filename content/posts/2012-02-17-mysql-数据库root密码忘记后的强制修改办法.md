---
title: mysql 数据库root密码忘记后的强制修改办法
author: 阿辉
date: 2012-02-17T09:35:00+00:00
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
第一：首先要把mysqld停止
```
service mysqld stop
```

第二：启动mysql，但是要跳过权限表
```
/usr/local/mysql/bin/mysqld_safe –skip-grant-tables &
```

第三：进去mysql，并修改密码
```
mysql -u root
mysql>use mysql;
mysql>update set user password=password("new_pass") where user="root";
mysql>flush privileges;
mysql>q
```
<!--more-->
第四：重新启动mysql，正常进入。