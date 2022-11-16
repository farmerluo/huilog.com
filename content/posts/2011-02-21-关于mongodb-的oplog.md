---
title: 关于mongodb 的Oplog
author: 阿辉
date: 2011-02-21T10:29:00+00:00
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
mongodb的Replication是通过一个日志来存储写操作的，这个日志就叫做Oplog。

在默认情况下,对于64位的mongodb,oplogs都相当大-可能是5%的磁盘空间。通常而言,这是一种合理的设置。可以通过mongod --oplogSize来改变Oplog的日志大小。

Oplog的collectio为：
```
local.oplog.$main for master/slave replication;
local.oplog.rs for replica sets
```
如 master/slave replication：
<!--more-->
```
> use local               
switched to db local
> db.oplog.$main.help()
DBCollection help
        db.oplog.$main.find().help() - show DBCursor help
        db.oplog.$main.count()
        db.oplog.$main.dataSize()
        db.oplog.$main.distinct( key ) - eg. db.oplog.$main.distinct( 'x' )
        db.oplog.$main.drop() drop the collection
        db.oplog.$main.dropIndex(name)
        db.oplog.$main.dropIndexes()
        db.oplog.$main.ensureIndex(keypattern,options) - options should be an object with these possible fields: name, unique, dropDups
        db.oplog.$main.reIndex()
        db.oplog.$main.find( [query] , [fields]) - first parameter is an optional query filter. second parameter is optional set of fields to return.
                                           e.g. db.oplog.$main.find( { x : 77 } , { name : 1 , x : 1 } )
        db.oplog.$main.find(...).count()
        db.oplog.$main.find(...).limit(n)
        db.oplog.$main.find(...).skip(n)
        db.oplog.$main.find(...).sort(...)
        db.oplog.$main.findOne([query])
        db.oplog.$main.findAndModify( { update : ... , remove : bool [, query: {}, sort: {}, 'new': false] } )
        db.oplog.$main.getDB() get DB object associated with collection
        db.oplog.$main.getIndexes()
        db.oplog.$main.group( { key : ..., initial: ..., reduce : ...[, cond: ...] } )
        db.oplog.$main.mapReduce( mapFunction , reduceFunction , <optional params> )
        db.oplog.$main.remove(query)
        db.oplog.$main.renameCollection( newName , ) renames the collection.
        db.oplog.$main.runCommand( name , <options> ) runs a db command with the given name where the first param is the collection name

        db.oplog.$main.save(obj)
        db.oplog.$main.stats()
        db.oplog.$main.storageSize() - includes free space allocated to this collection
        db.oplog.$main.totalIndexSize() - size in bytes of all the indexes
        db.oplog.$main.totalSize() - storage allocated for all data and indexes
        db.oplog.$main.update(query, object[, upsert_bool, multi_bool])
        db.oplog.$main.validate() - SLOW
        db.oplog.$main.getShardVersion() - only for use with sharding
        
> db.oplog.$main.find()
{ "ts" : { "t" : 1294582140000, "i" : 14 }, "op" : "d", "ns" : "mixi_top_city.building_90", "b" : true, "o" : { "_id" : "6380690_441_30_29" } }
{ "ts" : { "t" : 1294582140000, "i" : 15 }, "op" : "i", "ns" : "mixi_top_city.building_90", "o" : { "_id" : "6380690_441_24_24", "uid" : "6380690", "x" : 24, "y" : 24, "pos" : 1, "btime" : 1294154452, "ntime" : 1294154452, "bid" : 71, "extprop" : 0, "status" : 0, "ucid" : 441 } }
{ "ts" : { "t" : 1294582140000, "i" : 16 }, "op" : "u", "ns" : "mixi_top_city.building_64", "o2" : { "_id" : "16702364_459_14_14" }, "o" : { "$set" : { "status" : 1 } } }
{ "ts" : { "t" : 1294582140000, "i" : 17 }, "op" : "u", "ns" : "mixi_top_city.building_08", "o2" : { "_id" : "6223408_391_30_28" }, "o" : { "$set" : { "ntime" : 1294582321 } } }
{ "ts" : { "t" : 1294582140000, "i" : 18 }, "op" : "u", "ns" : "mixi_top_city.building_03", "o2" : { "_id" : "9882403_353_28_20" }, "o" : { "$set" : { "ntime" : 1294600141 } } }
{ "ts" : { "t" : 1294582140000, "i" : 19 }, "op" : "u", "ns" : "mixi_top_city.building_24", "o2" : { "_id" : "4162924_365_32_28" }, "o" : { "$set" : { "ntime" : 1294582321 } } }
{ "ts" : { "t" : 1294582141000, "i" : 1 }, "op" : "u", "ns" : "mixi_top_city.building_49", "o2" : { "_id" : "32797749_285_28_30" }, "o" : { "$set" : { "ntime" : 1294583341 } } }
{ "ts" : { "t" : 1294582141000, "i" : 2 }, "op" : "u", "ns" : "mixi_top_city.building_50", "o2" : { "_id" : "33768850_425_28_32" }, "o" : { "$set" : { "ntime" : 1294582561 } } }
{ "ts" : { "t" : 1294582141000, "i" : 3 }, "op" : "u", "ns" : "mixi_top_city.building_35", "o2" : { "_id" : "28235635_333_28_36" }, "o" : { "$set" : { "ntime" : 1294582741 } } }
{ "ts" : { "t" : 1294582141000, "i" : 4 }, "op" : "u", "ns" : "mixi_top_city.building_04", "o2" : { "_id" : "25178304_3_32_28" }, "o" : { "$set" : { "ntime" : 1294594141 } } }
{ "ts" : { "t" : 1294582141000, "i" : 5 }, "op" : "u", "ns" : "mixi_top_city.building_18", "o2" : { "_id" : "7304918_445_32_26" }, "o" : { "$set" : { "ntime" : 1294582321 } } }
{ "ts" : { "t" : 1294582141000, "i" : 6 }, "op" : "u", "ns" : "mixi_top_city.building_93", "o2" : { "_id" : "5003293_453_20_24" }, "o" : { "$set" : { "status" : 1 } } }
{ "ts" : { "t" : 1294582141000, "i" : 7 }, "op" : "u", "ns" : "mixi_top_city.building_59", "o2" : { "_id" : "19601459_485_28_30" }, "o" : { "$set" : { "ntime" : 1294582741 } } }
{ "ts" : { "t" : 1294582141000, "i" : 8 }, "op" : "u", "ns" : "mixi_top_city.building_47", "o2" : { "_id" : "23744647_273_22_46" }, "o" : { "$set" : { "ntime" : 1294582741 } } }
{ "ts" : { "t" : 1294582141000, "i" : 9 }, "op" : "u", "ns" : "mixi_top_city.building_50", "o2" : { "_id" : "3549050_451_20_30" }, "o" : { "$set" : { "ntime" : 1294583041 } } }
{ "ts" : { "t" : 1294582141000, "i" : 10 }, "op" : "d", "ns" : "mixi_top_city.building_77", "b" : true, "o" : { "_id" : "8577977_215_44_38" } }
{ "ts" : { "t" : 1294582141000, "i" : 11 }, "op" : "i", "ns" : "mixi_top_city.building_77", "o" : { "_id" : "8577977_215_44_38", "uid" : "8577977", "x" : 44, "y" : 38, "pos" : 1, "btime" : 1293955498, "ntime" : 1294486420, "bid" : 18, "extprop" : 0, "status" : 0, "ucid" : 215 } }
{ "ts" : { "t" : 1294582141000, "i" : 12 }, "op" : "u", "ns" : "mixi_top_city.building_89", "o2" : { "_id" : "21405489_541_20_24" }, "o" : { "$set" : { "status" : 1 } } }
{ "ts" : { "t" : 1294582141000, "i" : 13 }, "op" : "u", "ns" : "mixi_top_city.building_60", "o2" : { "_id" : "6479060_395_16_32" }, "o" : { "$set" : { "ntime" : 1294582321 } } }
{ "ts" : { "t" : 1294582141000, "i" : 14 }, "op" : "u", "ns" : "mixi_top_city.building_38", "o2" : { "_id" : "12696438_1037_28_40" }, "o" : { "$set" : { "ntime" : 1294583042 } } }
has more
>
```
Oplog日志中：

ts:Timestamp 这个操作的时间戳

op:operation 操作
```
i – insert
d – delete
u – update
c – command
n – no-op
```
ns:Namespace也就是操作的collection name

o:Document


查看master的Oplog信息：
```
[root@tc-03 cacti]# mongo
MongoDB shell version: 1.6.4
connecting to: test
> db.printReplicationInfo();
configured oplog size:   7503.049113600001MB
log length start to end: 3566227secs (990.62hrs)
oplog first event time:  Tue Jan 11 2011 12:17:03 GMT+0900 (KST)
oplog last event time:   Mon Feb 21 2011 18:54:10 GMT+0900 (KST)
now:                     Mon Feb 21 2011 18:54:10 GMT+0900 (KST)
```

查看slave的同步状态：
```
> db.printSlaveReplicationInfo()
source:   192.168.8.173
         syncedTo: Mon Feb 21 2011 18:55:19 GMT+0900 (KST)
                 = 31secs ago (0.01hrs)
```
 