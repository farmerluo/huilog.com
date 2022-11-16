---
title: 用Nagios监控Dell服务器硬件状况
author: 阿辉
date: 2011-12-22T14:29:00+00:00
categories:
- Nagios
tags:
- Nagios
keywords:
- Nagios
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
Dell有一套监控硬件的软件，Linux/windows都可以监控。

官方网址：http://linux.dell.com/repo/hardware/

安装方法（centos linux 5.7 x64）：

被监控服务器：

1) 增加dell的yum库
```bash
wget -q -O - http://linux.dell.com/repo/hardware/OMSA_6.5.2/bootstrap.cgi | bash
```
 

2) 安装srvadmin
```bash
Installing OpenManage Server Administrator
yum install srvadmin-all
```
 <!--more-->

3)安装firmware-tools，这个也可以不装，升级bios这类的用的。
Installing firmware-tools to manage BIOS and firmware updates
```bash
yum install dell_ft_install
```

4) 启动srvadmin：
```bash
/opt/dell/srvadmin/sbin/srvadmin-services.sh start
```
可以把上面的命令加入/etc/rc.local内开机启动。

5) 重启snmpd(假定您已经配置好了snmpd):
```bash
service snmpd restart
```

监控端：

1）安装perl的库:
```bash
yum install perl-Net-SNMP perl-Config-Tiny perl-Crypt-Rijndael
```

2) 安装nagios插件openmanage(需先配置好nagios server):
```bash
wget http://folk.uio.no/trondham/software/files/nagios-plugins-openmanage-3.7.3-1.el5.x86_64.rpm
```

3) 修改配置文件：
```bash
vim /etc/nagios/objects/commands.cfg
define command {
        command_name    check_openmanage
        command_line    /usr/lib64/nagios/plugins/check_openmanage -H $HOSTADDRESS$
}

vim /etc/nagios/objects/hosts.cfg
define service{
    use                             generic-service         ; Name of service template to use
    hostgroup_name                  all_servers
    service_description             Dell OMSA
    check_command                   check_openmanage
    normal_check_interval           10
    }
```
 

4) 重启nagios
```bash
service nagios reload
```

完成。

也可以手工用脚本调试：
```bash
[root@web ~]# /usr/lib64/nagios/plugins/check_openmanage -H 192.168.2.4
OK - System: 'PowerEdge R510 II', SN: 'XXXXXXX', 48 GB ram (6 dimms), 2 logical drives, 8 physical drives
```
 
如果报错：
```
 SNMP CRITICAL: No response from remote host '10.1.2.3'
```
或：
```
 ERROR: (SNMP) OpenManage is not installed or is not working correctly
```

首先确认/etc/snmpd.conf内是否有以下这行：
```
# Allow Systems Management Data Engine SNMP to connect to snmpd using SMUX
smuxpeer .1.3.6.1.4.1.674.10892.1
```
一般安装srvadmin的时候会自动加上的，如果没找到手工加上。

再检查被监控端的服务器上是否有启动snmpd和srvadmin，没有的话启动起来：
```bash
service snmpd restart

/opt/dell/srvadmin/sbin/srvadmin-services.sh restart
```

参考：
http://folk.uio.no/trondham/software/check_openmanage.html