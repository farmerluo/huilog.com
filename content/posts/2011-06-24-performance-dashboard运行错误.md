---
title: Performance Dashboard运行错误
author: 阿辉
date: 2011-06-24T10:09:00+00:00
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
Microsoft SQL Server 2005 Performance Dashboard Reports 是一组运行于 SQL Server 2005 SP2 Management Studio 及更高版本中的 Reporting Services 报表文件。他几乎统计了sql server 2005各个方面的报表，如cpu,内存，IO，未使用索引的查询等等，通过这些报表，SQL Server 用户可以更容易确定性能问题发生的时间以及可能存在的原因。这样可以更快地解决问题。

下载地址：

http://www.microsoft.com/download/en/details.aspx?displaylang=en&id=22602

今天运行 Performance Dashboard报了一个错，之前所有安装过程都是挺正常的，但执行“performance_dashboard_main.rdl”报表时出现“两个datetime 列的差别导致了运行时溢出。”错误。

<!--more-->

解决办法：

1) 修改setup.sql脚本中的procedure MS_PerfDashboard.usp_Main_GetSessionInfo的内容中的`“convert(bigint, datediff(ms, login_time, getdate()))”`为`“convert(bigint,datediff(dd,login_time,getdate()))*24*60*60*1000”`

2) 将recent_cpu.rdl中的`“convert(bigint, datediff(ms, login_time, getdate()))”`都换成“convert(bigint,datediff(dd,login_time,getdate()))*24*60*60*1000”`就可以了 

然后再执行setup.sql重新安装，再次运行Performance Dashboard，没有错误了。
