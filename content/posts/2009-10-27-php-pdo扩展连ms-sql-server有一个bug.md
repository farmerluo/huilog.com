---
title: php pdo扩展连ms sql server有一个Bug
author: 阿辉
date: 2009-10-27T17:40:00+00:00
categories:
- Php
tags:
- Php
keywords:
- Php
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
大部分时候都能连上，但每天会有一些连接不上的情况，会报下面的错误：

`PHP Fatal error:  Uncaught exception ‘PDOException’ with message ‘SQLSTATE[] (null) (severity 0)’`

或：

`PHP Fatal error:  Uncaught exception ‘PDOException’ with message ‘SQLSTATE[01002] Adaptive Server connection failed (severity 9)`

PHP 5.2.9 及 PHP 5.2.11都有这个问题，Bug目前还没有解决。

<!--more-->

准备用php的ms sql server扩展试试。

BUG信息如下：

http://bugs.php.net/bug.php?id=48539

http://bugs.php.net/bug.php?id=49344
