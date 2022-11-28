---
title: Sql Server SQL调优环境配置
author: 阿辉
date: 2009-12-07T16:19:00+00:00
categories:
- Sql Server
tags:
- Sql Server
keywords:
- Sql Server
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
- 显示有关由Transact-SQL 语句运行的时间的信息
`set statistics time on`

- 关闭有关由Transact-SQL 语句运行的时间的信息
`set statistics time off`

- 显示有关由Transact-SQL 语句生成的磁盘活动量的信息
`SET STATISTICS IO ON`

- 关闭有关由Transact-SQL 语句生成的磁盘活动量的信息
`SET STATISTICS IO OFF`

<!--more-->

- 显示[返回有关语句执行情况的详细信息，并估计语句对资源的需求]
`SET SHOWPLAN_ALL  ON`

- 关闭[返回有关语句执行情况的详细信息，并估计语句对资源的需求]
`SET SHOWPLAN_ALL  OFF`