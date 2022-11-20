---
title: linux IO监控工具iotop
author: 阿辉
date: 2010-01-31T11:24:00+00:00
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
如果你的linux kernel版本大于2.6.20,可以使用iotop这个工具，象windows 2008的任务管理器一样看到各个进程的IO读写。非常不错。

主页：http://guichaz.free.fr/iotop/

介绍：

Iotop is a Python program with a top like UI used to show of behalf of which
process is the I/O going on. It requires Python >= 2.5 (or Python >= 2.4 with
the ctypes module) and a Linux kernel >= 2.6.20 with the TASK_DELAY_ACCT and
TASK_IO_ACCOUNTING options enabled.

<!--more-->

如果python小于2.6，需要安装ctypes模块。
```
yum install python-ctypes
```
就行了。

可惜centos 5.4的内核是2.6.18。