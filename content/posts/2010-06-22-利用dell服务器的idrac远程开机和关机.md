---
title: 利用Dell服务器的iDRAC远程开机和关机
author: 阿辉
date: 2010-06-22T16:11:00+00:00
categories:
- 硬件
tags:
- Linux
- 硬件
keywords:
- Linux
- 硬件
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
基本上所有的Dell服务器都带有iDRAC功能，利用它可以完成远程开机和关机等操作，非常方便，得少跑多少机房呀。哈哈。

iDRAC配置：

有两个方法可以配置

1） 服务器开机的时候有个提示Ctrl+E进去配置iDRAC，按Ctrl+E进去，配置好IP地址，Netmask，用户，密码等。

2）安装Dell openManage Server Administrator，进入Dell openManage Server Administrator，在系统->主系统机箱->远程访问内也可以配置。

<!--more-->

iDRAC使用：

用浏览器打开刚刚配好的IP地址，输入用户名和密码，就可以进入 Integrated Dell Remote Access Controller 了。

里面大部分都是查看系统部件状态的，只有电源管理有点用，可以远程开关机。

![/wp-content/uploads/baiduhi/df6ec3ce04db0b3e92457e92.jpg](/wp-content/uploads/baiduhi/df6ec3ce04db0b3e92457e92.jpg)

另一个功能也非常不错，可以看上次崩溃屏幕和引导捕获的录像，不过需要本机安装jre才能看，如图：

![/wp-content/uploads/baiduhi/918a367ad0edbdd02f73b34d.jpg](/wp-content/uploads/baiduhi/918a367ad0edbdd02f73b34d.jpg)
