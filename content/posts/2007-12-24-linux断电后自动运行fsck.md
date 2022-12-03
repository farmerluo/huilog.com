---
title: Linux断电后自动运行fsck
author: 阿辉
date: 2007-12-24T14:36:00+00:00
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
输入命令： `cat -n /etc/rc.d/rc.sysinit`

显示如下：
```
echo ”Your system appears to have shut down uncleanly”
AUTOFSCK_TIMEOUT=5
[ -f /etc/sysconfig/autofsck ] && . /etc/sysconfig/autofsck
if [ “AUTOFSCK_DEF_CHECK” = “yes” ]; then
         AUTOFSCK_OPT=-f
fi
```
<!--more-->
可见与`/etc/sysconfig/autofsck`文件相关，可在bash进行如下操作修改：
```
touch /etc/sysconfig/autofsck
echo   AUTOFSCK_DEF_CHECK=yes >> /etc/sysconfig/autofsck
```
即创建autofsck文件，并将设定内容添加进去。

以后如果机器断电后出现的是
`Press N within 5 seconds to not force file system integrity check…`
5秒后就自动 fsck 了