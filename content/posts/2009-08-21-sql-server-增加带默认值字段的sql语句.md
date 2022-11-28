---
title: sql server 增加带默认值字段的sql语句
author: 阿辉
date: 2009-08-21T16:19:00+00:00
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
sql server 增加带默认值字段的sql语句,sql server比较特别，需要分开写。呵呵
```sql
BEGIN TRANSACTION
GO
ALTER TABLE dbo.gl_gemblitz_users ADD
uAppUser varchar(1000) NULL,
uAppUserInfo text NULL,
uFaceInfo varchar(50) NULL
GO
ALTER TABLE dbo.gl_gemblitz_users ADD CONSTRAINT
DF_gl_gemblitz_users_uAppUser DEFAULT (‘’) FOR uAppUser
GO
ALTER TABLE dbo.gl_gemblitz_users ADD CONSTRAINT
DF_gl_gemblitz_users_uAppUserInfo DEFAULT (‘’) FOR uAppUserInfo
GO
ALTER TABLE dbo.gl_gemblitz_users ADD CONSTRAINT
DF_gl_gemblitz_users_uFaceInfo DEFAULT (‘’) FOR uFaceInfo
GO
COMMIT

update dbo.gl_gemblitz_users set uAppUser=’’,uAppUserInfo=’’,uFaceInfo=’’
```
<!--more-->