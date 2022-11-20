---
title: 在Centos linux 5.x上安装Dell OpenManage
author: 阿辉
date: 2010-06-22T18:05:00+00:00
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
Dell OpenManage可以查看dell服务器各个部件的运行状态，并且有配置存储，远程管理等功能，对远程管理服务器非常有用，之前只在windows 2003上安装过，在windows下安装起来很简单，今天因为要配置服务器的iDARC，又不能重启服务器，所以就在centos linux系统上安装了dell OpenManage，碰到了点麻烦，在这里记录下：
```bash
wget http://ftp.us.dell.com/sysman/OM_6.1.0_ManNode_A00.tar.gz
tar xvzf OM_6.1.0_ManNode_A00.tar.gz
cd linux/supportscripts
sh srvadmin-install.sh -x
```
提示：Unsupported Operating System. Can not proceed….
<!--more-->
因为dell 只支持redhat Enterprise Linux，看了下脚本，是以开发的项目名来区分Linux版本的，Centos linux 5.x是RHEL5重新编译的，在redhat-release内加上RHEL5的项目名Tikanga就行了。做如下更改：
```bash
vi /etc/redhat-release
CentOS release 5.5 (Final)  Tikanga
```
再运行：

```bash
sh srvadmin-install.sh -x
Installing the selected packages.

warning: srvadmin-cm-6.1.0-648.i386.rpm: Header V3 DSA signature: NOKEY, key ID 23b66a9d
Preparing…                ########################################### [100%]
1:srvadmin-omilcore      ########################################### [  6%]
To start all installed services without a reboot,
enter the following command:  srvadmin-services.sh  start
2:srvadmin-omcommon      ########################################### [ 12%]
3:srvadmin-hapi          ########################################### [ 18%]
4:srvadmin-syscheck      ########################################### [ 24%]
5:srvadmin-deng          ########################################### [ 29%]
6:srvadmin-omacore       ########################################### [ 35%]
7:srvadmin-isvc          ########################################### [ 41%]
8:srvadmin-omauth        ########################################### [ 47%]
9:srvadmin-wsmanclient   ########################################### [ 53%]
10:srvadmin-idrac-componen########################################### [ 59%]
11:srvadmin-jre           ########################################### [ 65%]
12:srvadmin-cm            ########################################### [ 71%]
13:srvadmin-idracadm      ########################################### [ 76%]
14:srvadmin-idracdrsc     ########################################### [ 82%]
15:srvadmin-iws           ########################################### [ 88%]
16:srvadmin-omhip         ########################################### [ 94%]
17:srvadmin-storage       ########################################### [100%]
```
安装通过。

运行Dell OpenManage:
```bash
srvadmin-services.sh  start

Starting mptctl:
Waiting for mptctl driver registration to complete:
[  OK  ]

Starting Systems Management Device Drivers:
Starting dell_rbu:                                         [  OK  ]
Starting ipmi driver: Already started                      [  OK  ]
Starting Systems Management Data Engine:
Starting dsm_sa_datamgr32d:                                [  OK  ]
Starting dsm_sa_eventmgr32d:                               [  OK  ]
Starting dsm_sa_snmp32d:                                   [  OK  ]
Starting DSM SA Shared Services:                           [  OK  ]

Starting DSM SA Connection Service:                        [  OK  ]
```
全是OK，说明启动OpenManage成功，OpenManage监听的端口为1311，在浏览器内打开URL：https://ip:1311/，输入系统的用户名密码进入。里面看到的功能和Windows平台上的基本一样。


停止openManage的命令为：
```bash
srvadmin-services.sh stop
Shutting down DSM SA Shared Services:                      [  OK  ]
Shutting down DSM SA Connection Service:                   [  OK  ]
Stopping Systems Management Data Engine:
Stopping dsm_sa_snmp32d:                                   [  OK  ]
Stopping dsm_sa_eventmgr32d:                               [  OK  ]
Stopping dsm_sa_datamgr32d:                                [  OK  ]
Stopping Systems Management Device Drivers:
Stopping dell_rbu:                                         [  OK  ]
```
配置开机自启动：
```bash
srvadmin-services.sh  enable
```
关闭开机自启动：
```bash
srvadmin-services.sh disable
```

另外还有一种yum的方式安装OpenMagage，很简单:
```bash
wget -q -O - http://linux.dell.com/repo/hardware/OMSA_6.2/bootstrap.cgi | bash
yum install srvadmin-all
```