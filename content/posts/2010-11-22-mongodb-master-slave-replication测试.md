---
title: MongoDB master-slave replication测试
author: 阿辉
date: 2010-11-22T16:01:00+00:00
categories:
- Mongodb
tags:
- Mongodb
keywords:
- Mongodb
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
我们有项目用到了MongoDB，在正式运营时数据需要做实时备份，而目前资源也有限，只有两台机器用于MongoDB,所以选用MongoDB的Master-Slave Replication来实现实时备份，其实我是比较看好[Replica Sets](http://www.mongodb.org/display/DOCS/Replica+Sets)的，因为其可以实现自动切换，但我目前因为机器少只能做罢。

两台机器一台为Master(192.168.1.173),一台为Slave(192.168.1.174)，系统环境为:centos linux 5.5。
<!--more-->
1）在两台机器上都安装MongoDB

我们直接用官方编译好的rpm包

先增加mongodb的rpm库：
```bash
vi /etc/yum.repos.d/10gen.repo

[10gen]
name=10gen Repository
baseurl=http://downloads.mongodb.org/distros/centos/5.4/os/x86_64/
gpgcheck=0

# 再yum安装：

yum -y install mongo-stable mongo-stable-server
```

2)配置Master:

```bash
vi /etc/mongod.conf

# 增加下面几行：
rest = true
master = true

# 启动：
service mongod start
```

3)配置Slave:
```bash
vi /etc/mongod.conf

# 增加下面几行：

rest = true
slave = true
source = 192.168.1.174
autoresync = true

# 启动：
service mongod start
```

4)查看和测试

查看相关的状态:`db.printReplicationInfo()`

查看是否为Master：
`db.isMaster();`

测试：
在Master上增加一个Collection
`db.createCollection(“mycoll2”, {capped:true, size:100000})`

然后再在Slave上查看
`show collections`

如果能看到刚刚新增的mycoll2,说明成功了。


4）故障测试

A. Slave机器出了问题怎么办？

按照上面的3)中的步骤，重做一遍就是了。

B. Master机器出问题怎么办？

如果Master机器挂了，那么我们可以先把Slave改成Master让其提供服务：
```bash
# 在Slave上先停止mongod：

service mongod stop

# 再删除本地数据库，因为slave的相关信息存在这里面了。

cd /var/lib/mongo
rm -rf local.*

# 在配置文件内把slave改成master:
vi /etc/mongod.conf

# 删掉下面几行：
slave = true
source = 192.168.1.174
autoresync = true

# 增加:
master = true

# 最后再启动mongod：

service mongod start
```