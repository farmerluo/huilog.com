---
title: Sql Server 批量增加表索引的T-SQL
author: 阿辉
date: 2009-09-24T13:47:00+00:00
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
Sql Server 内有一批表，需增加一个索引，表名为`ct_user_jewe_00 - ct_user_jewe_99`，编写如下T-SQL完成：

<!--more-->
```sql
declare @i int
declare @table varchar(100)
declare @index varchar(100)
declare @sql nvarchar(1000)
SET @i=0
WHILE @i<=99
BEGIN
BEGIN TRANSACTION
IF @i < 10
BEGIN
SET @table=’dbo.ct_user_jewe_0’+CONVERT(char(5),@i)
SET @index=’IX_ct_user_jewe_0’+CONVERT(char(5),@i)
END
ELSE
BEGIN
SET @table=’dbo.ct_user_jewe_’+CONVERT(char(5),@i)
SET @index=’IX_ct_user_jewe_’+CONVERT(char(5),@i)
END
SET @sql=’CREATE NONCLUSTERED INDEX ‘+@index+’ ON ‘+@table
SET @sql=@sql+’ ( uId ) WITH( STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]’
PRINT @sql
execute sp_executesql @sql
SET @i=@i+1
COMMIT
END
```