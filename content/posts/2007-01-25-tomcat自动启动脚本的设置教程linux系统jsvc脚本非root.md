---
title: tomcat自动启动脚本的设置教程(Linux系统jsvc脚本非root)
author: 阿辉
date: 2007-01-25T16:10:00+00:00
categories:
- Tomcat
tags:
- Tomcat
keywords:
- Tomcat
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
如果直接就说启动脚本可能有些人会搞不明白。我刚开始也是有点乱，后来才看明白。现在就从头说一下吧。

1. 安装J2SDK(我的是下载后放在/root里面的下)
```bash
#cd
#./jdk-1_5_0_09-linux-i586.bin  　#翻到最后输入yes
#mv jdk1.5.0_09 /usr/local/java   #移动文件夹jdk1.5.0_09到/usr/local/里面并改名为java
```

2. 安装tomcat
```bash
#cd
#tar xvfz apache-tomcat-5.5.20.tar.gz
#mv apache-tomcat-5.5.20 /usr/local/tomcat  
```
移动文件夹apache-tomcat-5.5.20 到/usr/local/里面并改名为tomcat

<!--more-->

3. 设置环境变量
```bash
#vi /etc/profile  
# 在profile最后面加下以下内容

  export JAVA_HOME=/usr/local/java
  export PATH=$PATH:$JAVA_HOME/bin
  export CLASSPATH=.:$JAVA_HOME/jre/lib:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
  export TOMCAT_HOME=/usr/local/tomcat
  export CATALINA_HOME=/usr/local/tomcat
  export CLASSPATH=$CLASSPATH:$CATALINA_HOME/common/lib

#source /etc/profile   # 使环境变量生效
```

4. 安装jsvc
```bash
#cd /usr/local/tomcat/bin
#tar xvfz jsvc.tar.gz
#cd jsvc-src
#sh support/buildconf.sh
#chmod 755 configure
#./configure --with-java=/usr/local/java     # (改成你的JDK的位置)
#make
```

5. 添加脚本让tomcat自动启动
```bash
#useradd tomcat5  # 添加用户
#roupadd tomcat　　# 添加组
#usermod -G tomcat tomcat5　#　把tomcat5加入tomcat组
#chown -R tomcat5 /usr/local/tomcat　　# 设置用户tomcat5对tomcat的权限
#cp /usr/local/tomcat/bin/jsvc-src/native/Tomcat5.sh /etc/init.d/tomcat  # 移动文件tomcat5.sh到/etc/init.d/里面并改名为tomcat
#chmod 755 /etc/init.d/tomcat
#vi /etc/init.d/tomcat
```

我自己的内容如下，大家根据自己的情况修改
```bash
#!/bin/sh
#
# Startup Script for Tomcat5
#
# chkconfig: 345 88 14
# description: Tomcat Daemon
# processname: jsvc
# pidfile: /var/run/jsvc.pid
# config:
#
# Source function library.
. /etc/rc.d/init.d/functions
#
prog=tomcat
#
JAVA_HOME=/usr/local/java
CATALINA_HOME=/usr/local/tomcat
DAEMON_HOME=/usr/local/tomcat/bin
#TOMCAT_USER=tomcat5
TOMCAT_USER=tomcat5

# for multi instances adapt those lines.
TMP_DIR=/var/tmp
PID_FILE=/var/run/jsvc.pid
CATALINA_BASE=/usr/local/tomcat

CATALINA_OPTS=
CLASSPATH=
$JAVA_HOME/lib/tools.jar:
$CATALINA_HOME/bin/commons-daemon.jar:
$CATALINA_HOME/bin/bootstrap.jar

case "$1" in
  start)
    #
    # Start Tomcat
    #
    $DAEMON_HOME/jsvc-src/jsvc
    -user $TOMCAT_USER
    -home $JAVA_HOME
    -Dcatalina.home=$CATALINA_HOME
    -Dcatalina.base=$CATALINA_BASE
    -Djava.io.tmpdir=$TMP_DIR
    -wait 10
    -pidfile $PID_FILE
    -outfile $CATALINA_HOME/logs/catalina.out
    -errfile '&1'
    $CATALINA_OPTS
    -cp $CLASSPATH
    org.apache.catalina.startup.Bootstrap
    #
    # To get a verbose JVM
    #-verbose
    # To get a debug of jsvc.
    #-debug
    exit $?
    ;;

  stop)
    #
    # Stop Tomcat
    #
    $DAEMON_HOME/jsvc-src/jsvc
    -stop
    -pidfile $PID_FILE
    org.apache.catalina.startup.Bootstrap
    exit $?
    ;;

  *)
    echo "Usage tomcat.sh start/stop"
    exit 1;;
esac
```

测试tomcat能不能启动
```bash
#service tomcat start   
#chkconfig tomcat on 　
#chkconfig --list tomcat
```

重新启动linux试下吧：）
我在centos4.4试用通过。重起了１０次没有发现问题