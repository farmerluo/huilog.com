---
title: 使用php或python连接DB2的配置
author: 阿辉
date: 2014-08-19T03:22:35+00:00
categories:
- Db2
tags:
- Python
- PHP
- Db2
keywords:
- Python
- PHP
- Db2
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
最近写的一个程序需要连接一个DB2的数据库上去，配置环境的时候走了不少弯路，看了N多文档，搞了近两天，终于搞好了，在这里记录下：

先说明：系统是centos linux 5.x,64位，系统上的php是5.3，python的版本我有安装python 2.6的。

1)安装IBM Data Server Driver Package (DS Driver)

下载地址：https://www-304.ibm.com/support/docview.wss?uid=swg27016878

我下载下来的包名是：v10.5fp3_linuxx64_dsdriver.tar.gz
```bash
tar xvzf v10.5fp3_linuxx64_dsdriver.tar.gz
mkdir /opt/ibm
cp dsdriver /opt/ibm/
cd /opt/ibm/dsdriver./installDSDriver
```

<!--more-->

2)配置DS Driver：

cd /opt/ibm/dsdriver/cfg
vim db2cli.ini
```ini
[db2]
hostname=172.22.1.10
port=50110
database=msdb
uid=db2inst1
pwd=passwd
autocommit=0
```
上面[db2]是dsn，其它的就不用解释了。保存，退出。再执行：
```bash
[root@fft-vm-new-2 cfg]# /opt/ibm/dsdriver/bin/db2cli writecfg add -dsn db2 -database msdb -host 172.22.1.10 -port 50110

===============================================================================
db2cli writecfg completed successfully.
===============================================================================
```
可以看到db2dsdriver.cfg文件已经生成了：
```xml
[root@fft-vm-new-2 cfg]# cat db2dsdriver.cfg
<?xml version="1.0" encoding="UTF-8" standalone="no" ?>
<configuration>
<dsncollection>
    <dsn alias="db2" host="172.22.1.10" name="msdb" port="50110"/>
</dsncollection>

<databases>
    <database host="172.22.1.10" name="msdb" port="50110"/>
</databases>

</configuration>
```
配置环境变量：

DS Driver安装后已经生成了一个配置文件在/opt/ibm/dsdriver目录：

vim db2profile
```bash
# Generic PATH and library path settings
export PATH="/opt/ibm/dsdriver/./bin":"$PATH"
export PATH="/opt/ibm/dsdriver/./adm":"$PATH"

export LD_LIBRARY_PATH="/opt/ibm/dsdriver/./lib":"$LD_LIBRARY_PATH"

# Environment variables for sqlj and JDBC/JCC drivers
export CLASSPATH="/opt/ibm/dsdriver/./java/db2jcc.jar":"$CLASSPATH"
export CLASSPATH="/opt/ibm/dsdriver/./java/sqlj.zip":"$CLASSPATH"

# Environment variables for open source drivers
export IBM_DB_DIR="/opt/ibm/dsdriver/."
export IBM_DB_HOME="/opt/ibm/dsdriver/."  #这条手动加一下，DS Driver生成时是没有这条的，python 的扩展编译时需要这个环境变量
export IBM_DB_LIB="/opt/ibm/dsdriver/./lib"
export IBM_DB_INCLUDE="/opt/ibm/dsdriver/./include"
export DB2_HOME="/opt/ibm/dsdriver/./include"
export DB2LIB="/opt/ibm/dsdriver/./lib"

# Environment variables for CLPPlus utility
export CLASSPATH="/opt/ibm/dsdriver/./tools/clpplus.jar":"$CLASSPATH"
export CLASSPATH="/opt/ibm/dsdriver/./tools/jline-0.9.93.jar":"$CLASSPATH"
export CLASSPATH="/opt/ibm/dsdriver/./rdf/lib/antlr-3.3-java.jar":"$CLASSPATH"
```
保存这个文件，再修改：

vim /etc/profile
在倒数第三行加上：`source /opt/ibm/dsdriver/db2profile`保存，退出。并重新登陆linux服务器，让环境变量生效。

3）配置php模块
```bash
cd /opt/ibm/dsdriver/php/php64
cp ibm_db2_5.3.6_nts.so pdo_ibm_5.3.6_nts.so /usr/lib64/php/modules/
```
新增下面两个配置文件：
```bash
[root@fft-vm-new-2 dsdriver]# vim /etc/php.d/ibm_db2.ini
extension=ibm_db2_5.3.6_nts.so
[root@fft-vm-new-2 dsdriver]# vim /etc/php.d/pdo_ibm.ini
extension=pdo_ibm_5.3.6_nts.so
```
重启php，让其生效:
`service httpd restart`
再用php -i就可以看到这两个PHP扩展。

写段代码测试：
vim db2test.php
```php
<?php
$dsn="db2";
$conn = db2_connect($dsn, '', '');

if ($conn) {
    echo "Connection succeeded.";
    db2_close($conn);
} else {
    echo "Connection failed.";
}
?>
```
执行：
```
[root@fft-vm-new-2 ~]# php db2test.php
Connection succeeded.
```
4）python的扩展安装

python的模块最低支持2.6的版本，所以安装前确定你的系统上有python 2.6版本。
模块名叫ibm_db,下面几个网址可以参考：
https://pypi.python.org/pypi/ibm_db/
https://code.google.com/p/ibm-db/wiki/
https://code.google.com/p/ibm-db/wiki/APIs

你可以使用easy_install-2.6 ibm_db安装这个模块，但是模块的代码是存放在code.google.com上的，我用easy_install时下载不下来，所以我是从https://pypi.python.org/pypi/ibm_db/下载源码进行安装的。
```bash
wget https://pypi.python.org/packages/source/i/ibm_db/ibm_db-2.0.5.tar.gz#md5=73ed86f4cf423fc608db95403ba988e4
tar xvzf ibm_db-2.0.5.tar.gz
cd ibm_db-2.0.5
python26 setup.py install
```
OK,这样不报错的话，模块就安装好了，在这里我就不写测试脚本了，使用这个模块可以参考上面几个链接的API。

补充：

有时候用php脚本的db2_connect函数连接db2时会报错：
```
Aug 19 15:08:33 fft-vm-new-2 php: PHP Warning:  PHP Startup: Unable to load dynamic library '/usr/lib64/php/modules/ibm_db2_5.3.6_nts.so' - libdb2.so.1: cannot open shared object file: No such file or directory in Unknown on line 0
Aug 19 15:08:33 fft-vm-new-2 php: PHP Warning:  PHP Startup: Unable to load dynamic library '/usr/lib64/php/modules/pdo_ibm_5.3.6_nts.so' - libdb2.so.1: cannot open shared object file: No such file or directory in Unknown on line 0
Aug 19 15:08:33 fft-vm-new-2 php: PHP Fatal error:  Call to undefined function db2_connect() in /usr/lib/zabbix/alertscripts/sendsms.php on line 108
Aug 19 15:08:33 fft-vm-new-2 php: PHP Warning:  PHP Startup: Unable to load dynamic library '/usr/lib64/php/modules/ibm_db2_5.3.6_nts.so' - libdb2.so.1: cannot open shared object file: No such file or directory in Unknown on line 0
Aug 19 15:08:33 fft-vm-new-2 php: PHP Warning:  PHP Startup: Unable to load dynamic library '/usr/lib64/php/modules/pdo_ibm_5.3.6_nts.so' - libdb2.so.1: cannot open shared object file: No such file or directory in Unknown on line 0
Aug 19 15:08:33 fft-vm-new-2 php: PHP Fatal error:  Call to undefined function db2_connect() in /usr/lib/zabbix/alertscripts/sendsms.php on line 108
```
解决办法：

我们看看libdb2.so.1这个文件是不是存在？
```bash
[root@fft-vm-new-2 ~]# find /opt -name 'libdb2.so.1'
/opt/ibm/dsdriver/lib32/libdb2.so.1
/opt/ibm/dsdriver/lib/libdb2.so.1
/opt/ibm/odbc_cli/clidriver/lib/libdb2.so.1
```
文件是在的，只是找不到，加入到ldconfig：
`vim /etc/ld.so.conf`
在最后加入下面这行：
`/opt/ibm/dsdriver/lib/`
保存，然后执行：
`ldconfig`
让其生效，再跑PHP脚本，发现已经不报错了。

 