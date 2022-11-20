---
title: 用nagios监控mongodb
author: 阿辉
date: 2010-12-03T16:21:00+00:00
categories:
- Mongodb
tags:
- Mongodb
- Nagios
keywords:
- Mongodb
- Nagios
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
网上已经有人写好了mongodb的nagios监控脚本，参考：
https://github.com/mzupan/nagios-plugin-mongodb/blob/master/README.md

1) 先安装git
```bash
yum install git
```

2) 下载脚本
```bash
cd /etc/nagios/command

git clone git://github.com/mzupan/nagios-plugin-mongodb.git
cd nagios-plugin-mongodb/
chmod 755 check_mongodb.py
```
如果执行报下面的错误：
```bash
# ./check_mongodb.py --help
need to install pymongo
```
需要安装pymongo：
```bash
git clone git://github.com/mongodb/mongo-python-driver.git pymongo
cd pymongo/
python setup.py install
```

<!--more-->

3) 修改nagios配置，加入这个命令
```bash
vi objects/commands.cfg
# 'check_mongodb' command definition
define command {
        command_name    check_mongodb
        command_line    /etc/nagios/command/nagios-plugin-mongodb/check_mongodb.py -H $HOSTADDRESS$ -A $ARG1$ -P $ARG2$ -W $ARG3$ -C $ARG4$
        }
```

4) 加入mongo监控的配置
```bash
vi hosts.cfg
define service{
        use                             generic-service         ; Name of service template to use
        host_name                       monodb_host
        service_description             mongodb
        check_command                   check_mongodb!connect!27017!10!30
        notifications_enabled           1
        }
```   

5) 没错的话，重载nagios就行了。
```bash
nagios -v /etc/nagios/nagios.cfg
service nagios reload
```	