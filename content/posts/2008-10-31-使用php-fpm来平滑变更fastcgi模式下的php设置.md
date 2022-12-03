---
title: 使用php-fpm来平滑变更FastCGI模式下的php设置
author: 阿辉
date: 2008-10-31T10:32:00+00:00
categories:
- Nginx
tags:
- Nginx
keywords:
- Nginx
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
在使用FastCGI方式运行php的时候，如果我们改变了php.ini的设置，就得重新启动php的fastcgi守护程序。如果你的系统负载 比较大的话，这个重启过程或许会让你的系统中断服务一段时间。php-fpm就是为了解决这个问题而诞生的，它可以在php的fastcgi进程不中断的 情况下重新加载你改动过的php.ini。
而且php-fpm可以不用再依赖其它的fastcgi启动器，比如lighttpd的spawn-fcgi，对于我来说终于可以摆脱lighttpd的影子了。

还等什么，开始吧！

我的php版本是5.2.6，先到官网下载与php版本对应的php-fpm补丁:![PHP-FPM](http://php-fpm.anight.org/)

<!--more-->

假设：php源代码目录在：`/usr/local/src/php-5.2.6`，php-fpm下载到了`/usr/local/src`
```
cd /usr/local/src
gzip -cd php-5.2.6-fpm-0.5.9.diff.gz | patch -d php-5.2.6 -p1
```
补丁打好以后，编译php的时候增加了下面几个参数：
```
–enable-fpm 激活fastcgi模式的fpm支持
–with-fpm-conf php-fpm的配置文件（默认是PREFIX/etc/php-fpm.conf）
–with-fpm-log php-fpm的日志文件（默认是PREFIX/logs/php-fpm.log）
–with-fpm-pid php-fpm的pid文件（默认是PREFIX/logs/php-fpm.pid）
```
编译的时候–enable-fpm当然是必须的，其它几项可以根据你自己的情况自行调整。
编译参数：
```
cd /usr/local/src/php-5.2.6
./configure --enable-fastcgi --enable-fpm --prefix=/usr/local/php5 --with-config-file-path=/usr/local/php5/etc --THE-OTHERS
make
make install
```
其中–THE-OTHERS代表了其它的一些php编译参数，这里就略去了。

稍等片刻，等php编译并安装好，下面开始配置php-fpm。

`vi /usr/local/php5/etc/php-fpm.conf`

php-fpm.conf是一个xml格式的纯文本文件，具体细节可以自己打开看看，基本上一看就明白了。在这里特别注意一下这几个配置字段：

`127.0.0.1:9000`

这个表示php的fastcgi进程监听的ip地址以及端口

nobody
nobody

表示php的fastcgi进程以什么用户以及用户组来运行

0
是否显示php错误信息

5
最大的子进程数目

下面运行php-fpm：

`/usr/local/php5/bin/php-cgi --fpm`

现在php的fastcgi进程就已经在后台运行，并监听127.0.0.1的9000端口。
可以用ps和netstat来看看结果：
```
ps aux | grep php-cgi
netstat -tpl | grep php-cgi
```
安装好了php-fpm，那么它是怎么来达到我们最初的目的的呢？
很幸运，php-fpm自己就给我们准备了一个程序来控制fastcgi进程，这个文件在$PREFIX/sbin/php-fpm

运行一下：

`/usr/local/php5/sbin/php-fpm`
该程序有如下参数：
```
start 启动php的fastcgi进程
stop 强制终止php的fastcgi进程
quit 平滑终止php的fastcgi进程
restart 重启php的fastcgi进程
reload 重新加载php的php.ini
logrotate 重新启用log文件
```
也就是说，在修改了php.ini之后，我们可以使用

`/usr/local/php5/sbin/php-fpm reload`

这样，就保持了在php的fastcgi进程持续运行的状态下，又重新加载了php.ini。

后话：这个文件可以稍作修改，让它成为CentOS/Redhat的服务，减少一些管理上的开销。
