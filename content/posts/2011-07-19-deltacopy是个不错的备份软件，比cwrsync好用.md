---
title: Deltacopy是个不错的备份软件，比cwrsync好用
author: 阿辉
date: 2011-07-19T16:17:00+00:00
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
Deltacopy和cwrsync一样，都是基于大名鼎鼎的同步工具rsync。但是Deltacopy比cwrsync要多了几个管理工具，好用多了。

官方网站：http://www.aboutmyip.com/AboutMyXApp/DeltaCopy.jsp

Deltacopy是开源软件，基于GPL v3,可以免费使用，无需注册。

用官网下解绿色版，解压后会看到几个执行文件：DCServce.exe,DeltaC.exe,DeltaS.exe,DSetup.exe,rsync.exe,ssh.exe。

DeltaS.exe是服务端程序

DSetup.exe同上

DeltaC.exe是客户端程序

DCServce.exe是安装windows服务的程序

rsync.exe和ssh.exe就不用介绍了，同步和备份就靠它们了。

<!--more-->

使用方法：

1) 配置服务端

我们需要先配置服务端，Deltacopy服务端可以理解成一个专用的备份服务器，客户端会把需要备份的东西推送到备份服务器上。

打开DeltaS.exe，会看到下图，配置一个虚拟目录，客户端就会把备份文件同步到这个目录内了。

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg)

然后把它注册成windows服务：

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/675d3bc7b1077eb9d0006055.jpg)

启动服务，服务端就配置好了。

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/af0ba901c731e8631c958355.jpg)

2）客户端配置

新增一个prfile,把刚刚配置的服务器信息填上去，用add folder增加一个需要备份的目录。

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/e091918f3cd58e9f503d9255.jpg)

options内可以配置备份的一些参数  
![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/1dd4f21f0c30ed06f724e455.jpg)

点击Modify Schedule,配置一下计划任务。配置好后就可以不用客户端了，定时自动会跑备份程序。

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/2141d5397e39bc963b87ce55.jpg)

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/baa659826e8e56c70df4d255.jpg)

点Run Now,可以立即执行备份。  
![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/92a7e5dd1128168a77c63855.jpg)

下图为备份状态：

![/wp-content/uploads/baiduhi/abacb1352fc004eea71e1255.jpg](/wp-content/uploads/baiduhi/5ec60923d26ed02b93580755.jpg)
