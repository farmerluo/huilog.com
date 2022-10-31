---
title: ActiveMQ 配置为每个队列一个kahaDB
author: 阿辉
date: 2019-08-13T09:09:41+00:00
categories:
- ActiveMQ
tags:
- ActiveMQ
keywords:
- ActiveMQ
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

ActiveMQ 使用KahaDB存储时，默认的配置是所有队列都存在一个KahaDB内。使用久了会发现一个问题，就是DB文件越来越大，几十上百G。主要原因是当队列内只要有一条消息没有被消费掉，那么ActiveMQ是不会清理KahaDB的文件的。

为此，我们可以加一个过滤器，配置成每个队列一个KahaDB，来缓解这个问题，配置如下：
```xml
        <persistenceAdapter>
           <mKahaDB directory="${activemq.data}/kahadb">
            <filteredPersistenceAdapters>
                  <!-- kahaDB per destinations -->
              <filteredKahaDB perDestination="true">
                <persistenceAdapter>
                    <kahaDB ignoreMissingJournalfiles="true"  checkForCorruptJournalFiles="true"  checksumJournalFiles="true" />
                </persistenceAdapter>
              </filteredKahaDB>
             </filteredPersistenceAdapters>
           </mKahaDB>
        </persistenceAdapter>
```

<!--more-->


老的配置通常为：
```xml
        <persistenceAdapter>
            <kahaDB directory="${activemq.data}/kahadb"
                    ignoreMissingJournalfiles="true"
                    checkForCorruptJournalFiles="true"
                    checksumJournalFiles="true" /> 
        </persistenceAdapter>
```

另外顺便提一下发现的另一个问题，通常我们都会配置一个存储空间的限制，比如：
```xml
          <systemUsage>
            <systemUsage>
                <memoryUsage>
                    <memoryUsage percentOfJvmHeap="70" />
                </memoryUsage>
                <storeUsage>
                    <storeUsage limit="70 gb"/>
                </storeUsage>
                <tempUsage>
                    <tempUsage limit="10 gb"/>
                </tempUsage>
            </systemUsage>
        </systemUsage>
```
意思是存储空间的限制为70GB，但是我有一次发现，当磁盘空间还有很多时，日志提示空间满了。
```bash
2019-08-13 12:58:56,945 | INFO  | Usage(default:store:queue://Consumer.expand.VirtualTopic.memberActiveMessage:store) percentUsage=99%, usage=24906188032, limit=24884771791, percentUsageMinDelta=1%;Parent:Usage(default:store) percentUsage=100%, usage=24906188032, limit=24884771791, percentUsageMinDelta=1%: Persistent store is Full, 100% of 24884771791. Stopping producer (ID:sh-saas-o2o-member-online-02-45191-1565359975591-1:23037:1:1) to prevent flooding queue://Consumer.expand.VirtualTopic.memberActiveMessage. See http://activemq.apache.org/producer-flow-control.html for more info (blocking for: 1563s) | org.apache.activemq.broker.region.Queue | ActiveMQ Transport: tcp:///10.16.149.233:11266@61616
```
上面这条日志居然提示存储的限制是24G左右，已经使用满了，而我们配置的是70G。而磁盘空间目前也有很多未使用。但是这些队列已经不能再进来了。

现在我唯一能想到的可能是：当时activeMQ启动时，磁盘空间只有24G左右，虽然我们在配置内配置的是70G，但是activeMQ启动时会检查磁盘的剩余空间，当磁盘的剩余空间小于配置文件内的storeUsage时，就会把磁盘的剩余空间当做限制值。即便是以后，我们通过清理磁盘空间后，磁盘可用空间大于这个限制值时，限制值也不会变，还是启动时检测到的那个值。