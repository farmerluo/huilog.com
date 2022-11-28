---
title: 在不停止mysql复制主服务器的情况下，配置一个mysql复制从服务器
author: 阿辉
date: 2009-04-08T14:56:00+00:00
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
首先配置好从服务器的my.cnf，启动mysql。

1.在主服务器上，备份数据库

`mysqldump -uroot -p111111 –master-data -B game ucenter uchome ihompy > ihompy.sql`

备份数据库，注意必需带–master-data参数。

2.在从服务器上的mysql内：

`stop slave`

停止从服务器的数据更新。

<!--more-->

3.导入备份数据库到从服务器内：

`mysql -h 10.11.12.26 -uroot -p111111 < ihompy.sql`

4.看导入是否成功

比对`show slave status`和`head -n 50 ihompy.sql`内的MASTER_LOG_FILE及MASTER_LOG_POS是否相同。相同说明导入OK。

5.启动从服务器：

`start slave`

再用`show slave status`看看是否有报错，MASTER_LOG_POS是否有变化，无报错，有变化说明配置成功了。