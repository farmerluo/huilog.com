---
title: infobright IEE 3.4.2 试用
author: 阿辉
date: 2010-09-17T10:37:00+00:00
categories:
- Infobright
tags:
- Infobright
keywords:
- Infobright
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
Infobright是一个与MySQL集成的数据仓库软件，一般用于数据分析。

Infobright的优点在于基于列存储，所以无需建索引，因为本身就相当于索引了。在存储数据库非常大的情况下，查询性能也可以非常好。

但是也有限制，官方建议用LOAD DATA INFILE的方式导入数据，因为社区版Infobright只能使用“LOAD DATA INFILE”的方式导入数据，不支持INSERT、UPDATE、DELETE，而企业版虽然支持insrt，但速度也很慢，不建议使用。

Infobright分为社区版(ICE)和企业版(IEE)，社区版有些限制，近日试用了infobright IEE 3.4.2。
<!--more-->

1) 安装

在http://www.infobright.com/注册一个帐号，可以下载infobright IEE 3.4.2。并可以下载一个30天的试用License Key 。

我下载的是IEE v3.4.2 64-Bit RPM (Red Hat Enterprise 5.x and CentOS 5.x) 。得到的是一个rpm包，直接安装就行：
```
rpm -ivh infobright-3.4.2-p2-x86_64-eval.rpm
```
然后把Licentse Key 复制到infobright目录内：
```
cp iblicense-luo_hui-42275088.lic /usr/local/infobright-3.4.2-x86_64/
```
安装就完成了。

启动：`service mysqld-ib start`
停止：`service mysqld-ib stop`

2)导入数据

我从一个sql server内找了一个比较大的表，有几百万条记录。将其导出到一个aaa.csv文件上，上传到服务器。

转换下格式：`dos2unix aaa.csv`

infobright使用上和mysql基本一样。

建表：
```bash
/usr/local/infobright-3.4.2-x86_64/bin/mysql -uroot -p -Dtest
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 5
Server version: 5.1.40-log build number (revision)=IB_3.4.2_DPN128_9237(iee_eval - commercial)

Type ‘help;’ or ‘h’ for help. Type ‘c’ to clear the current input statement.

mysql> CREATE TABLE IF NOT EXISTS testib (
->   lId int(11) NOT NULL,
->   lType int(11) NOT NULL,
->   luId varchar(20)  NULL,
->   uId varchar(20)  NULL,
->   lRank int(11) NULL,
->   lMoney int(11) NULL,
->   lStone int(11) NULL,
->   lExp int(11)  NULL,
->   uFloor int(11)  NULL,
->   lTime int(11) NOT NULL
-> ) ENGINE=BRIGHTHOUSE DEFAULT CHARSET=utf8;
```
用mysql命令导入数据：
```bash
bin/mysql -S /tmp/mysql-ib.sock -uroot -p -D test –skip-column-names -e “LOAD DATA INFILE ‘/usr/local/infobright-3.4.2-x86_64/aaa.csv’ INTO TABLE testib FIELDS TERMINATED BY ‘,’ ESCAPED BY ‘’ LINES TERMINATED BY ‘n’;”
```

3) 测试

在另一台mysql上也导入了一份数据，对比测试infobright与mysql的查询速度。

mysql上testib表的结构为：
```sql
CREATE TABLE IF NOT EXISTS testib (
lId int(11) NOT NULL,
lType int(11) NOT NULL,
luId varchar(20) default NULL,
uId varchar(20) default NULL,
lRank int(11) default NULL,
lMoney int(11) default NULL,
lStone int(11) default NULL,
lExp int(11) default NULL,
uFloor int(11) default NULL,
lTime int(11) NOT NULL,
PRIMARY KEY  (lId),
KEY luId (luId),
KEY lStone (lStone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

a) 百万级记录的测试

mysql查询速度：
```sql
mysql> select count() from testib;
+———-+
| count() |
+———-+
|  4580404 |
+———-+
1 row in set (2.03 sec)

mysql> SELECT lType, count() AS total FROM testib GROUP BY lType order by lType DESC;
+——-+———+
| lType | total   |
+——-+———+
|     6 |   30020 |
|     5 |   65952 |
|     4 |  340863 |
|     3 | 2200985 |
|     2 |  229015 |
|     1 | 1713569 |
+——-+———+
6 rows in set (2.03 sec)
```


infobright 查询速度：
```sql
mysql> select count() from testib;
+———-+
| count() |
+———-+
|  4580404 |
+———-+
1 row in set (0.00 sec)

mysql> SELECT lType, count() AS total FROM testib GROUP BY lType order by lType DESC;
+——-+———+
| lType | total   |
+——-+———+
|     6 |   30020 |
|     5 |   65952 |
|     4 |  340863 |
|     3 | 2200985 |
|     2 |  229015 |
|     1 | 1713569 |
+——-+———+
6 rows in set (0.63 sec)
```

另外测试在mysql一个索引字段上的查询：
`SELECT lStone, count() AS total FROM testib GROUP BY lStone order by lStone DESC;`

mysql 结果：
`2291 rows in set (2.54 sec)`

infobright结果：
`2291 rows in set (0.65 sec)`


b) 亿记录级别测试

将这表的记录加大到1.3亿左右，再看测试结果：

infobright：
```sql
[root@mysqltest infobright-3.4.2-x86_64]# bin/mysql -uroot -p -Dtest
Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 45
Server version: 5.1.40-log build number (revision)=IB_3.4.2_DPN128_9237(iee_eval - commercial)

Type ‘help;’ or ‘h’ for help. Type ‘c’ to clear the current input statement.

mysql> select count(*) from testib;
+———–+
| count()  |
+———–+
| 137412120 |
+———–+
1 row in set (0.00 sec)

mysql> SELECT lType, count() AS total FROM testib GROUP BY lType order by lType DESC;
+——-+———-+
| lType | total    |
+——-+———-+
|     6 |   900600 |
|     5 |  1978560 |
|     4 | 10225890 |
|     3 | 66029550 |
|     2 |  6870450 |
|     1 | 51407070 |
+——-+———-+
6 rows in set (19.45 sec)

mysql> SELECT lStone, count() AS total FROM testib GROUP BY lStone order by lStone DESC;
2291 rows in set (22.70 sec)

mysql> SELECT sum(lStone) AS total,lType FROM testib GROUP BY lType order by total DESC;
+————+——-+
| total      | lType |
+————+——-+
| 5844824280 |     1 |
| 1488257820 |     2 |
|   29491980 |     5 |
|          0 |     3 |
|          0 |     4 |
|          0 |     6 |
+————+——-+
6 rows in set (28.79 sec)
```

mysql：
```sql
[root@x64test ~]# mysql -uroot -p -Dtest
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 6012
Server version: 5.0.77 Source distribution

Type ‘help;’ or ‘h’ for help. Type ‘c’ to clear the buffer.

mysql> select count() from testib;
+———–+
| count()  |
+———–+
| 137412120 |
+———–+
1 row in set (18 min 44.82 sec)

mysql> SELECT lType, count() AS total FROM testib GROUP BY lType order by lType DESC;
+——-+———-+
| lType | total    |
+——-+———-+
|     6 |   900600 |
|     5 |  1978560 |
|     4 | 10225890 |
|     3 | 66029550 |
|     2 |  6870450 |
|     1 | 51407070 |
+——-+———-+
6 rows in set (30 min 12.97 sec)

mysql> SELECT lStone, count() AS total FROM testib GROUP BY lStone order by lStone DESC;
2291 rows in set (10 min 45.11 sec)

mysql> SELECT sum(lStone) AS total,lType FROM testib GROUP BY lType order by total DESC;
+————+——-+
| total      | lType |
+————+——-+
| 5844824280 |     1 |
| 1488257820 |     2 |
|   29491980 |     5 |
|          0 |     4 |
|          0 |     6 |
|          0 |     3 |
+————+——-+
6 rows in set (25 min 6.96 sec)
```
从测试结果看，infobright的查询速度的确是比mysql要快很多。