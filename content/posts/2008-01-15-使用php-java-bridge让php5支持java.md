---
title: 使用php-java-bridge让PHP5支持java
author: 阿辉
date: 2008-01-15T17:59:00+00:00
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
环境：
       服务器是64位的。
       centos linux 5.0 (x86_64)
       系统自带apache 2.2及php5.x

所需安装包(latest version)：
jdk-1_5_0_12-linux-amd64.bin （http://java.sun.com/j2se/1.5.0/download.jsp）
php-java-bridge_5.0.0.tar.gz （http://php-java-bridge.sourceforge.net/）

<!--more-->

1.安装jdk-1_5_0_05

下载地址：http://java.sun.com/j2se/1.5.0/download.jsp
```bash
cp /path/to/ jdk-1_5_0_12-linux-amd64.bin /usr/local/
cd /usr/local
chmod +x jdk-1_5_0_12-linux-amd64.bin
./ jdk-1_5_0_12-linux-amd64.bin
ln -s jdk1.5.0_12 jdk
```

2.设置环境变量，java的运行需要设置一下环境变量。

在/etc/profile中设置如下参数：
```bash
JAVA_HOME=/usr/local/java
PATH=PATH:JAVA_HOME/bin
```
并且export它们。
修改原来的export语句为：

`export JAVA_HOME PATH USER LOGNAME MAIL HOSTNAME HISTSIZE INPUTRC`

若要立即生效，在shell下边依次执行一遍上边的语句。
输入java -version能看到版本信息，即安装java成功了。

3.安装php-java-bridge_5.0.0.tar.gz

下载地址: http://php-java-bridge.sourceforge.net/
```bash
tar php-java-bridge_5.0.0.tar.gz
cd php-java-bridge-5.0.0
（具体环境要求和安装请阅读INSTALL文档）
phpize
./configure –with-java=JAVA_HOME
make && make install
```
编辑php.ini文件
增加
```
[Java]
java.java_home=”/usr/local/java”
java.java=”/usr/local/java/jre/bin/java”
java.log_file=”/var/log/php-java-bridge.log”
java.classpath=”/usr/lib64/php/modules/JavaBridge.jar”
java.libpath=”/usr/lib64/php/modules”
extension_dir=”/usr/lib64/php/modules/“
extension=java.so
```
验证：

重启Apache ，用pstree查看，有`httpd—java—java—8*[java]`进程。

用命令行方式检测`echo ‘‘ | php | fgrep java`，应该返回字样有`java status => running`

通过Web方式查看phpinfo() ，存在Java小节。

在访问目录下创建java.php文件
```php
<?php
phpinfo();
print "nn";

v = new java("java.lang.System");
arr=v->getProperties();
foreach (arr as key => value) {
    print key . " -> " . $value . "\n";
}
?>
```

通过Web访问，能正确显示Java版本、操作系统、系统时间等信息，说明执行成功。