---
title: 配置访问列式数据库vertica的php环境
author: 阿辉
date: 2011-03-24T20:27:00+00:00
categories:
- Vertica
tags:
- Vertica
keywords:
- Vertica
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
vertica是一个和Sybase IQ,infobright类似的列式数据库，没有用过Sybase IQ，就和infobright 3.4的社区版做了简单对比测试，在同样一个一亿条记录的表中，下面的查询：
```sql
select count(*) from c_log group by level;
```
vertica花了5秒，infobright花了2分20秒。

测试表c_log的结构为：
```sql
CREATE TABLE c_log (
  uid varchar(20),
  cash integer,
  gold integer,
  level integer,
  rtime datetime,
  tid varchar(20),
  act varchar(50),
  item varchar(500),
  value integer,
  value2 integer,
  time datetime
);
```
<!--more-->

同时infobright的社区版是不支持insert语句的，只能用LOAD语句来插入数据，也就是说社区版不支持同时插入和查询，不过infobright商业版是支持的。vertica没有这个问题，当然，vertica也是商业的，需要钱买,在他们的官网可以申请30天的试用license。

php访问vertica不像访问infobright，infobright是兼容mysql协议的，直接用imysql或pdo_mysql就行。而vertica不兼容mysql协议，所以也就不能用php的mysql模块来访问了。

vertica网站上的数据库驱动只有odbc和jdbc的，jdbc是java用的，对php来说，那就只能用odbc来访问了。

在centos linux 5.x上用odbc，首先要安装unixODBC：
```
yum install unixODBC
```
然后到vertica官网下载odbc的驱动，注意要根据自己的平台来下载，我这边用的是64位的：
```bash
tar -xvzf vertica_4.0.24_odbc_3.5_unixodbc_x86_64_linux.tar.gz
cp include/verticaodbc.h /usr/include
cp lib64/* /usr/lib64
```
配置odbc：
```bash
vi /etc/odbc.ini

[ODBC Data Sources]
VerticaDSN = "vertica"

[VerticaDSN]
Description = VerticaDSN ODBC driver
Driver = /usr/lib64/libverticaodbc_unixodbc.so
Database = dbname
Servername = 192.168.1.10
UserName =
Password =
Port = 5433
ConnSettings =
Locale = en_GB
```

还要更改odbcinst.ini：
```ini
vi /etc/odbcinst.ini

[VerticaDSN]
Description = ODBC for VerticaDSN
Driver = /usr/lib64/libverticaodbc_unixodbc.so

[ODBC]
Threading = 1
```
改完后可以用
```
isql -v VerticaDSN username password
```
不测试一下，是否可以访问数据库，能访问的话，说明在linux下的odbc配置成功了。

下一步还需要安装php的odbc模块,odbc模块是php自带的，从php官网下载源码包，解压，然后：
```
cd ext/odbc
/usr/local/php5/bin/phpize
./configure --with-php-config=/usr/local/php5/bin/php-config
```
但是我安装php的odbc模块一直不成功，提示：
```
checking for Adabas support... cp: cannot stat `/usr/local/lib/odbclib.a': No such file or directory
configure: error: ODBC header file '/usr/local/incl/sqlext.h' not found!
```
试了各种办法和参数都还是这样，正好又看到还有一个pdo_odbc模块，就准备用pdo_odbc了：

编译安装php的pdo_odbc模块
```
yum install unixODBC-devel

cd ext/pdo_odbc
./configure --with-php-config=/usr/local/php5/bin/php-config    --with-pdo-odbc=unixODBC,/usr
make
make install
```
```
vi php.ini
```
加入一行：
```
extension = "pdo_odbc.so"
```
重启web服务就可以通过phpinfo()看到pdo_odbc模块了。

至于php怎么用pdo_odbc访问vertica？参照php手册吧。