---
title: Vertica数据库sql操作备忘
author: 阿辉
date: 2012-03-01T14:49:00+00:00
categories:
- Verticadb
tags:
- Verticadb
keywords:
- Verticadb
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
删除主键（Vertica数据库的主键值并不是唯一的）：
```sql
SELECT ANALYZE_CONSTRAINTS(‘fb_s.c_log’);
```
找到key名，再：
```sql
ALTER TABLE fb_s.c_log DROP CONSTRAINT C_PRIMARY;

SELECT ANALYZE_CONSTRAINTS(‘fb_s.user_info’);

ALTER TABLE fb_s.user_info DROP CONSTRAINT C_PRIMARY;
```
<!--more-->
建用户和SCHEMA ：
```sql
CREATE user fb_s_sql IDENTIFIED BY ‘password’;
CREATE SCHEMA fb_s_sql;
```

给权限：
```sql
GRANT ALL ON SCHEMA fb_s_sql TO fb_s_sql;
GRANT ALL ON SCHEMA fb_s TO fb_s_sql;

GRANT ALL ON TABLE fb_s_sql.sqllog TO fb_s_sql;
```

建表：
```sql
CREATE TABLE fb_s.c_log (
    uid int  NOT NULL,
    cash int,
    gold int,
    level int,
    rtime datetime,
    tid varchar(20),
    act varchar(50),
    item varchar(500),
    value int,
    value2 int,
    time datetime
);

CREATE TABLE fb_s.new_c_log (
  uid integer PRIMARY KEY NOT NULL,
  cash integer,
  gold integer,
  level integer,
  rtime datetime,
  tid varchar(20),
  act varchar(50),
  item varchar(500),
  value integer,
  value2 integer,
  time datetime NOT NULL
)
PARTITION BY EXTRACT(year FROM time)*100 + EXTRACT(month FROM time);
```
后一个是按time字段分区

增加及修改字段：
```sql
ALTER TABLE fb_s.c_logADD COLUMN value2 integer default 0;
ALTER TABLE fb_s.c_log  ALTER COLUMN duration SET DEFAULT 0;
ALTER TABLE fb_s.c_log  ALTER COLUMN mesg SET DEFAULT ”;
```

两表之间导数据：
```sql
insert into fb_s.c_log (uid,cash,gold,level,rtime,tid,act,item,value,value2,time)
(select * from fb_s.c_logbak);
```
两库之间导数据：

在源库导出：
```sql
vsql -d topcity -U dbadmin -w password -F ‘,’ -At -o fs_user_info.csv -c "SELECT * FROM fb_s.user_info;" &
vsql -d topcity -U dbadmin -w password -F ‘,’ -At -o fs_c_log.csv -c "SELECT * FROM fb_s.c_log;" &
```

目的库导入：
```sql
COPY fb_s.user_info  FROM ‘/opt/fs_user_info.csv’ EXCEPTIONS ‘/tmp/exp.log’ DELIMITER ‘,’;
COPY fb_s.c_log  FROM ‘/opt/fs_c_log.csv’ EXCEPTIONS ‘/tmp/exp.log’ DELIMITER ‘,’;
```