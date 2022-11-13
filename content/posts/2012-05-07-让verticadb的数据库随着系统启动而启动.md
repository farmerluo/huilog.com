---
title: 让verticadb的数据库随着系统启动而启动
author: 阿辉
date: 2012-05-07T09:34:00+00:00
categories:
- Verticadb
tags:
- Verticadb
keywords:
- Verticadb
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
要想让verticadb的数据库随着系统启动而启动，把下面这行加入到/etc/rc.local:
```
su - dbadmin -c '/opt/vertica/bin/adminTools -t start_db -d dbname -p password -i'
```

执行结果类似这样：
```
su - dbadmin -c '/opt/vertica/bin/adminTools -t start_db -d verticadb-p password -i' 

        Node Status: v_verticadb_node0001: (DOWN)
        Node Status: v_verticadb_node0001: (DOWN)
        Node Status: v_verticadb_node0001: (DOWN)
        Node Status: v_verticadb_node0001: (DOWN)
        Node Status: v_verticadb_node0001: (DOWN)
        Node Status: v_verticadb_node0001: (UP)
```
<!--more-->