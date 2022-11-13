---
title: centos 5上python 2.6版本的easy_install安装
author: 阿辉
date: 2014-08-18T09:00:49+00:00
categories:
- Python
tags:
- Python
- Linux
keywords:
- Python
- Linux
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
cenots linux 5.x上默认安装的python版本是2.4.3。现在很多python脚本和模块已经不支持python 2.4的版本了，想用那些模块就需要升级到python 2.6以上。但是centos linux 5.x上的很多软件如yum都是基于python 2.4开发的，把系统自带的python直接升级到python 2.6，会造成yum等这些都不能使用。所以不能这么干，得在一个操作系统上安装两个或多个python版本。

在epel的yum源上，已经有做好的python26包。先加上epel源，再直接 yum安装就行。

yum install python26 python26-devel

直接运行python26命令就是新版本的python,系统上的python命令还是2.4的版本。

<!--more-->

虽然现在系统上已经有了python 2.6了，但是要想安装python 2.6的模块还是有点麻烦。因为目前系统上的easy\_install还是python 2.4的，所以我们需要安装python 2.6的easy\_install，用他来安装python 2.6的模块。

下载easy_install,执行安装脚本：

```bash
wget http://peak.telecommunity.com/dist/ez_setup.py

python26 ez_setup.py
```

好了，现在系统上有3个easy_install：

```
[root@fft-vm-new-2 fftfo]# ll /usr/bin/easy_install*  
-rwxr-xr-x 1 root root 285 Aug 18 14:13 /usr/bin/easy_install  
-rwxr-xr-x 1 root root 288 May 25  2008 /usr/bin/easy_install-2.4  
-rwxr-xr-x 1 root root 293 Aug 18 14:13 /usr/bin/easy_install-2.6
```
现在可以安装python 2.6的模块了，如：

`easy_install-2.6 setuptools`