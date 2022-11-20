---
title: mysql over ssh 访问
author: 阿辉
date: 2010-12-03T18:04:00+00:00
categories:
- Mysql
tags:
- Mysql
keywords:
- Mysql
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
先来假设一个场景，在机房有几台服务器，分别是：

mysql:192.168.1.10
web:192.168.1.20
test:192.168.1.30

有如下限制：
1) test可以ssh到web,不能ssh到mysql
2) web不能ssh到mysql
3) web可以访问mysql
4) test不能访问mysql
5) mysql用户没有show databses权限，也就是用不了phpmyadmin
6) 在公司只能ssh到test

而现在你在公司，除了先ssh到test,再从test ssh到web,用命令行访问mysql,还有什么办法？
答案可能只有mysql over ssh tunnel了。

<!--more-->

需要打两次洞，test打一个到mysql的，通过web代理；还有就是从公司到test的。

配置步骤：
1）先让test能无密码(使用证书)ssh到web

在test上：

生成一个公私key：
```
# ssh-keygen -t dsa
```
一直回车，不要输密码
把公key 复制到web上
```bash
scp /.ssh/*.pub test@192.168.1.20:/
```

在web上：

```bash
cd /home/test
cat id_dsa.pub >> .ssh/authorized_keys
chmod 644 .ssh/authorized_keys
```
在test上测试，看ssh test@192.168.1.20是不是不用密码就能连过去了。

2) 在test上建立tunnel
```bash
# ssh -L3308:192.168.1.10:3306 -p 22 -N -t -x test@192.168.1.20
```
现在应该可以在test上面用`mysql -uuser -p -h127.0.0.1 -P 3308`访问数据库了。

3）在secureCRT上开一个tunnel

编辑连接到test主机的属性：

![/wp-content/uploads/baiduhi/1b61d7cade349a0af31fe7ba.jpg](/wp-content/uploads/baiduhi/1b61d7cade349a0af31fe7ba.jpg)
这样就建立了一个从你的机器到test服务器3308的tunnel。

现在你访问本机的3308端口就相当于访问192.168.1.10的3306端口，也就是访问mysql数据库。

4) 用MySQL-Front连接数据库

打开MySQL-Front,新建一个连接，主机为127.0.0.1,端口为3308：

![/wp-content/uploads/baiduhi/95905aaff72fdab7fbed509a.jpg](/wp-content/uploads/baiduhi/95905aaff72fdab7fbed509a.jpg)
