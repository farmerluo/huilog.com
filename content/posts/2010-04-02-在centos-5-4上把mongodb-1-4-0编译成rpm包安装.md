---
title: 在centos 5.4上把mongodb 1.4.0编译成rpm包安装
author: 阿辉
date: 2010-04-02T17:45:00+00:00
categories:
- Mongodb
tags:
- Mongodb
keywords:
- Mongodb
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
rpm包还是比较好管理和升级的，今天试着把mongodb 1.4.0做成rpm包。

网上的文章比较少，只有自己摸索着做了。

首先需要先做一些准备工作：

安装epel库，这一步可有可无。
```bash
wget http://download.fedora.redhat.com/pub/epel/5/x86_64/epel-release-5-3.noarch.rpm
rpm -ivh epel-release-5-3.noarch.rpm
```
<!--more-->
安装一些相关的rpm,编译或运行mongo需要这些：
```bash
yum install -y bzip2-devel python-devel libicu-devel chrpath zlib-devel nspr-devel readline-devel ncurses-devel boost-devel pcre-devel js-devel readline-devel git tcsh scons gcc-c++ glibc-devel js js-devel
```
还需要将centos 5.4上的boost从1.33升级到1.38,mongodb建议1.34以上，更好的支持高并发情况。如果不升级，编译的时候会有相关的警告信息。

这一块想了很多办法，从官网http://www.boost.org/上下载源码包编译成rpm没有成功，而且版本差异过大，担心会有问题，所以偷了一下懒，直接用网上的src.rpm了。

下载：
```bash
wget ftp://195.220.108.108/linux/sourceforge/g/project/gr/gridiron2/support-files/FC10%20source%20RPMs/boost-1.38.0-1.fc10.src.rpm
```
或：
```bash
wget http://mirror.transact.net.au/sourceforge/g/project/gr/gridiron2/support-files/FC10%20source%20RPMs/boost-1.38.0-1.fc10.src.rpm
```

安装：
`rpm -ivh boost-1.38.0-1.fc10.src.rpm`

打包：
`rpmbuild -ba /usr/src/redhat/SPECS/boost.spec`

用刚刚编译的rpm包升级本机的boost：
`rpm -Uvh /usr/src/redhat/RPMS/x86_64/boost-1.38.0-1.x86_64.rpm /usr/src/redhat/RPMS/x86_64/boost-devel-1.38.0-1.x86_64.rpm`

查看一下：
```bash
[root@x64test ~]# rpm -qa | grep boost
boost-devel-1.38.0-1
boost-1.38.0-1
```
OK了，都是1.38版本的。

还要重新编译pcre,自带的编译时没带–enable-unicode-properties参数，用自带的编译出来的mongdb启动时会提示：
```bash
[root@x64test ~]# mongod
warning: some regex utf8 things will not work.  pcre build doesn’t have –enable-unicode-properties
```

我们下载src.rpm包重新编译：
```bash
[root@x64test ~]# wget http://mirrors.sohu.com/centos/5.4/os/SRPMS/pcre-6.6-2.el5_1.7.src.rpm
[root@x64test ~]# rpm -ivh pcre-6.6-2.el5_1.7.src.rpm
[root@x64test ~]# vi /usr/src/redhat/SPECS/pcre.spec
%configure –enable-utf8
```
改成：
```bash
%configure –enable-utf8 –enable-unicode-properties

[root@x64test ~]# rpmbuild -ba /usr/src/redhat/SPECS/pcre.spec
[root@x64test ~]# rpm -Uvh /usr/src/redhat/RPMS/x86_64/pcre-6.6-2.7.x86_64.rpm
[root@x64test ~]# rpm -Uvh /usr/src/redhat/RPMS/x86_64/pcre-devel-6.6-2.7.x86_64.rpm
```

下面开始编译mongodb 1.4.0：

1) 先下载：
`wget http://downloads.mongodb.org/src/mongodb-src-r1.4.0.tar.gz`

2) 解压：
`tar -xvzf mongodb-src-r1.4.0.tar.gz`

3) 按mongo.spec定义的文件名和目录重新做一个压缩包出来：
```bash
mv mongodb-src-r1.4.0  mongo-1.4.0
tar -czf mongo-1.4.0.tar.gz mongo-1.4.0
```

4) copy到打包目录：
```bash
cp mongo-1.4.0/rpm/mongo.spec /usr/src/redhat/SPECS/
cp mongo-1.4.0.tar.gz /usr/src/redhat/SOURCES/
```

5) mongo.spec有个小bug,还得改下：
```bash
vi /usr/src/redhat/SPECS/mongo.spec
```
找到第71行，将：
```bash
/usr/sbin/useradd -M -r -U -d /var/lib/mongo -s /bin/false
-c mongod mongod > /dev/null 2>&1
```
换成：
```bash
/usr/sbin/useradd -M -r -d /var/lib/mongo -s /bin/false
-c mongod mongod > /dev/null 2>&1
```

或者:
```bash
/usr/sbin/useradd -M -r -d /var/lib/mongo -s /bin/false
-c mongod mongod > /dev/null 2>&1 || true
```
这是一个bug,useradd没有-U这个参数，小写的-u倒是有。不改这个编译完成后，mongo-server会安装不成功。

上面一个更改经过测试没有问题，但是需要确保安装的时候系统内没有mongod用户存在，如果这个用户存在的话还是会报错（rpm -Uvh肯定就不行了）。后面的更改是为解决这个问题的，不过没有测试过。


6) 开始编译打包：
```bash
rpmbuild -ba /usr/src/redhat/SPECS/mongo.spec
```

7) 安装：
```bash
cp /usr/src/redhat/RPMS/x86_64/mongo.rpm .
rpm -ivh mongo.rpm
```

安装完成。