---
title: elasticsearch重启恢复延迟的相关参数
author: 阿辉
date: 2019-10-31T08:53:05+00:00
categories:
- ElasticSearch
tags:
- ElasticSearch
keywords:
- ElasticSearch
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

下面这个参数是elasticsearch延迟分配的超时时间,不让集群认为节点失效而发起均衡。重启节点前可以加这个配置，减少平衡：

```bash
curl -XPUT 'http://127.0.0.1:9200/_all/_settings' -H 'Content-Type: application/json' -d '
{
  "settings": {
    "index.unassigned.node_left.delayed_timeout": "5m"
  }
}'
```

另外还有3个参数：
```
gateway.recover_after_nodes: 8
gateway.expected_nodes: 10
gateway.recover_after_time: 5m
```

<!--more-->

这意味着 Elasticsearch 会采取如下操作：
- 等待集群至少存在 8 个节点
- 等待 5 分钟，或者10 个节点上线后，才进行数据恢复，这取决于哪个条件先达到。

这三个设置可以在集群重启的时候避免过多的分片交换。这可能会让数据恢复从数个小时缩短为几秒钟。

注意：这些配置只能设置在 config/elasticsearch.yml 文件中或者是在命令行里（它们不能动态更新）它们只在整个集群重启的时候有实质性作用。

参考：这将阻止 Elasticsearch 在存在至少 8 个节点（数据节点或者 master 节点）之前进行数据恢复。 这个值的设定取决于个人喜好：整个集群提供服务之前你希望有多少个节点在线？这种情况下，我们设置为 8，这意味着至少要有 8 个节点，该集群才可用。