---
title: Drbd + heartbeat + mysql replication来构建mysql的高可用性
author: 阿辉
date: 2007-08-23T10:57:00+00:00
categories:
- Mysql
tags:
- Drbd
- Heartbeat
- Mysql
keywords:
- Drbd
- Heartbeat
- Mysql
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
A(M)[192.168.33.11192.168.43.11] ->B (Backup)[192.168.33.13192.168.43.13] ->C (M/S)[192.168.33.15192.168.43.15] -> s1、s2….

公用IP：192.168.33.100

本例可实现以下功能：

一、实现mysql replication

A(M)–>C(M/S)–>s1、s2、s3….

性能：降低A服务器的负载

扩展性：可扩展到20台slave服务器。

二、实现实时备份、安全可靠功能

利用drbd(号称网络RAID)将A服务器与C服务器的数据进行实时备份

如果仅A服务器down掉了，通过heartbeatB服务器则会自动切换，变成A服务器的角色.

如果仅C服务器down掉了，则需要在B服务器上，进行手动切换，变成C服务器的角色.

如果A和C服务器都down掉了，则需要改进本例，方可解决问题.(如：再增加一台backup机器)

<!--more-->

本例需要的软件包如下：
```
drbd-0.7.23-1.c4.x86_64.rpm
heartbeat-2.0.7-1.c4.x86_64.rpm
heartbeat-pils-2.0.7-1.c4.x86_64.rpm
heartbeat-stonith-2.0.7-1.c4.x86_64.rpm
kernel-module-drbd-2.6.9-42.0.10.ELsmp-0.7.23-1.el4.centos.x86_64.rpm
mysql-5.0.33.tar.gz
```

1. 分区：

在A、C机器分出/opt分区

在B机器上分出/opt1、/opt2分区

且大小相同/opt=/opt1=/op2

2. 安装mysql并进行mysql replication设定.
```bash
[mdbrw01 ~]#tar -zxf mysql-5.0.33.tar.gz
[mdbrw01 ~]#cd mysql-5.0.33
[mdbrw01 ~]#./configure –prefix=/usr/local/mysql –with-mysqld-ldflags=-all-static –with-mysqld-user=mysql –with-charset=cp932 –with-pthread CFLAGS=-O3 CXXFLAGS=-O3 CXX=gcc
[mdbrw01 ~]#make &&　make install
[mdbrw01 ~]#/usr/local/mysql/bin/mysql -A　-e “grant REPLICATION SLAVE on . to [email=slaver@]slaver@”%[/email]” Identified by”slave;FLUSH PRIVILEGES;”
[mdbrw01 ~]# /usr/local/mysql/bin/mysqladmin shutdown
[mdbrw01 ~]# mv /usr/local/mysql /opt
[mdbrw01 ~]#ln -s /opt/mysql /usr/local/mysql
```
replication设定比较简单，在此就不说明了.

3. 分别在A、B、C机器上安装以下软件
```
drbd-0.7.23-1.c4.x86_64.rpm
   kernel-module-drbd-2.6.9-42.0.10.ELsmp-0.7.23-1.el4.centos.x86_64.rpm
   heartbeat-pils-2.0.7-1.c4.x86_64.rpm
   heartbeat-stonith-2.0.7-1.c4.x86_64.rpm
   heartbeat-2.0.7-1.c4.x86_64.rpm
```
   (请按照顺序来进行安装)

==配置drbd===

在A服务器上:
```bash
[mdbrw01 ~]#vi /etc/drbd.conf
resource r0 {
   protocol C;
   startup {
degr-wfc-timeout 120;
    }
   disk {
on-io-error detach;
   }
   net {
   }
   syncer {
rate 10M;
group 1;
al-extents 257;
   }
   on mdbrw01{
device     /dev/drbd0;
disk    /dev/sda5;
address 192.168.43.11:7788;
meta-disk   internal;
   }
   on mdbbk01
device /dev/drbd0;
disk    /dev/sda5;
address 192.168.43.13:7788;
meta-disk internal;
   }
}
```

启动Drbd并设置为Primary:
```bash
[mdbrw01 ~]# /etc/init.d/drbd start
[mdbrw01 ~]#drbdadm – –do-what-I-say primary all
```

建立专用块及文件系统:
```bash
[mdbrw01 ~]#mknod /dev/drbd0 b 147 0
[mdbrw01 ~]#mkfs /dev/drbd0
```

在B服务器上:
```bash
[mdbrwbak ~]#vi /etc/drbd.conf
resource r0 {
   protocol C;
   startup {
degr-wfc-timeout 120;
   }
   disk {
on-io-error detach;
   }
   net {
   }
   syncer {
group 1;
al-extents 257;
   }
   on mdbrw01{
device     /dev/drbd0;
disk    /dev/sda5;
address 192.168.43.11:7788;
meta-disk   internal;
   }
   on mdbbk01{
device /dev/drbd0;
disk    /dev/sda5;
address 192.168.43.13:7788;
meta-disk internal;
   }
}
resource r1 {
   protocol C;
   incon-degr-cmd “echo ‘!DRBD! pri on incon-degr’ | wall ; sleep 60 ; halt -f”;
   startup {
wfc-timeout       0;
degr-wfc-timeout   120;
   }
   disk {
on-io-error detach;
   }
   net {
   }
   syncer {
rate 4M;
group 1;
   }
   on mdbrw02{
device     /dev/drbd1;
disk    /dev/sda5;
address 192.168.43.15:7789;
meta-disk   internal;
   }
   on mdbbk01 {
device     /dev/drbd1;
disk    /dev/sda6;
address 192.168.43.13:7789;
meta-disk   internal;
   }
}
```

启动Drbd:
`[mdbrwbak ~]# /etc/init.d/drbd start`

建立专用块及文件系统:
```bash
[mdbrwbak ~]#mknod /dev/drbd0 b 147 0
[mdbrwbak ~]#mknod /dev/drbd1 b 147 1
```

在C服务器上:
```bash
[mdbrw02 ~]#vi /etc/drbd.conf
resource r0 {
   protocol C;
   startup {
degr-wfc-timeout 120;
   }
   disk {
on-io-error detach;
   }
   net {
   }
   syncer {
rate 10M;
group 1;
al-extents 257;
   }
   on mdbrw02{
device     /dev/drbd1;
disk    /dev/sda5;
address 192.168.43.15:7789;
meta-disk   internal;
   }
   on mdbbk01{
device /dev/drbd1;
disk    /dev/sda6;
address 192.168.43.13:7789;
meta-disk internal;
   }
}
```

启动Drbd并设置为Primary:
```bash
[mdbrw02 ~]# /etc/init.d/drbd start
[mdbrw02 ~]#drbdadm – –do-what-I-say primary all
```

建立专用块及文件系统:
```bash
[mdbrw02 ~]#mknod /dev/drbd1 b 147 1
[mdbrw02 ~]#mkfs /dev/drbd1
```

==配置heartbeat==

分别在A、B服务器上创建authkeys和haresources文件
```bash
# vi /etc/ha.d/authkeys
auth 1
1 crc
#2 sha1 HI!
#3 md5 Hello!
[mdbrw01 ~]#chmod 600 /etc/ha.d/authkeys
[mdbrw01 ~]#vi /etc/ha.d/haresources
mdbrw01 IPaddr::192.168.33.100/24/eth1 mysql_umount mysql
[mdbrw01 ~]# cat /etc/ha.d/resource.d/mysql_umount
#!/bin/sh
unset LC_ALL; export LC_ALL
unset LANGUAGE; export LANGUAGE
prefix=/usr
exec_prefix=/usr
. /etc/ha.d/shellfuncs
case “1” in
‘start’)
/sbin/drbdadm – –do-what-I-say primary r0
#/sbin/drbdadm – –do-what-I-say primary all
/bin/mount /dev/drbd0 /opt
       ;;
‘pre-start’)
       ;;
‘post-start’)
       ;;
‘stop’)
/bin/umount /opt
/sbin/drbdadm   secondary r0  
#/sbin/drbdadm   secondary all   
;;
‘pre-stop’)
       ;;
‘post-stop’)
       ;;
*)
       echo “Usage: 0 { start | pre-start | post-start | stop | pre-stop | post-stop }”
       ;;
esac
exit 0
[mdbrw01 ~]# cat /etc/ha.d/resource.d/mysql
内容略..
[mdbrw01 ~]# cat /etc/ha.d/ha.cf
logfile /var/log/ha-log
logfacility     local0
keepalive 625ms
deadtime 5
warntime 1250ms
initdead 30
udpport 699
bcast eth1          # Linux
auto_failback off
node mdbrw01
node mdbbk01
ping   192.168.33.1
respawn hacluster /usr/lib64/heartbeat/ipfail
```
===启动heartbeat==
分别在A、B服务器上:
`[mdbrw01 ~]#etc/init.d/heartbeat start`

注意事项：

1.drbd.conf文件中每一个资源的配置需要相同.

2.在/opt mount状态下不能启动drbd

3.在drbd启动状态下添加、修改、删除drbd.conf文件中的资源，可能会导致drbd stop失败

4.A、B服务器上的ha.cf需要保持一致