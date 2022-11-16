---
title: 用deltacopy备份，出现File name too long (91)错误
author: 阿辉
date: 2011-07-20T10:52:00+00:00
categories:
- Windows
tags:
- Windows
keywords:
- Windows
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
出现File name too long (91)错误是因为老版本的rsync最大只支持255个字符（包括目录和文件名），超过255个字符就会报错。

解决办法是用一个更新版的rsync，在windows上的rsync只有Cygwin上有了，去Cygwin官网下载setup.exe，然后安装，把所有net都选中。安装完以后把下面文件copy到Deltacopy目录。
```
chmod.exe  
cygcrypto-0.9.8.dll  
cyggcc_s-1.dll  
cygiconv-2.dll  
cygintl-8.dll  
cygminires.dll  
cygpopt-0.dll  
cygwin1.dll  
cygz.dll  
rsync.exe  
ssh.exe
```
copy完后还要改一个配置文件deltacd.conf：

<!--more-->
在开头加上：
```
uid = 0  
gid = 0
```
然后重启deltacopy服务就行。

上面那些文件我打了个包，可以在这下载：

http://farmerluo.googlecode.com/files/cywin1.7.rar
