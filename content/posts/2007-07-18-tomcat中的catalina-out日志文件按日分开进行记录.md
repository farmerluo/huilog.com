---
title: tomcat中的catalina.out日志文件按日分开进行记录
author: 阿辉
date: 2007-07-18T15:18:00+00:00
categories:
- Tomcat
tags:
- Tomcat
keywords:
- Tomcat
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
在`bin/catalina.sh`文件里，原代码：
```bash
org.apache.catalina.startup.Bootstrap "$@" start       
"$CATALINA_BASE"/logs/catalina.out 2>&1 &
```
修改为：       
```bash
org.apache.catalina.startup.Bootstrap "$@" start       
2>&1 |/usr/sbin/cronolog "$CATALINA_BASE"/logs/catalina.out.%Y-%m-%d &
```
<!--more-->
其中/usr/sbin是日志轮询工具的目录。修改保存退出后重启tomcat即可生效。

shutdown.sh，再startup.sh后，catalina.out就变成了catalina.out.2005-11-25的形式。

这样做的好处是，catalina.out 文件不会很大，容易分析内容，也容易使用自动脚本每天定时查找其中的问题所在。