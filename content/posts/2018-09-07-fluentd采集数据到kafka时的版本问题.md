---
title: fluentd采集数据到kafka时的版本问题
author: 阿辉
date: 2018-09-07T06:21:52+00:00
tags:
- fluentd
- kafka
categories:
- Kubernetes
- fluentd
tags:
- Kubernetes
- fluentd
- kafka
keywords:
- Kubernetes
- fluentd
- kafka
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---
在使用fluentd采集数据到kafka时，一直不通，碰到了很多报错。
fluentd版本为：1.2.5
fluent-plugin-kafka版本为：0.7.8
kafka版本为：0.9
最开始碰到了这个报错：
```bash
2018-09-05 01:42:06 +0000 [warn]: fluent/log.rb:342:warn: Send exception occurred: unknown topic 
2018-09-05 01:42:06 +0000 [warn]: fluent/log.rb:342:warn: Exception Backtrace : /var/lib/gems/2.3.0/gems/ruby-kafka-0.6.8/lib/kafka/protocol/metadata_response.rb:141:in `partitions_for'
/var/lib/gems/2.3.0/gems/ruby-kafka-0.6.8/lib/kafka/cluster.rb:155:in `partitions_for'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:190:in `assign_partitions!'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:153:in `block in deliver_messages_with_retries'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:148:in `loop'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:148:in `deliver_messages_with_retries'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:102:in `deliver_messages'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/out_kafka2.rb:220:in `write'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin/output.rb:1110:in `try_flush'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin/output.rb:1389:in `flush_thread_run'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin/output.rb:444:in `block (2 levels) in start'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin_helper/thread.rb:78:in `block in thread_create'
2018-09-05 01:42:06 +0000 [info]: fluent/log.rb:322:info: initialized kafka producer: fluentd
2018-09-05 01:42:06 +0000 [debug]: fluent/log.rb:302:debug: taking back chunk for errors. chunk="57515e0ef787da843836cc864f9d1581"
2018-09-05 01:42:06 +0000 [warn]: fluent/log.rb:342:warn: failed to flush the buffer. retry_time=2 next_retry_seconds=2018-09-05 01:42:06 +0000 chunk="57515e0ef787da843836cc864f9d1581" error_class=Kafka::UnknownTopicOrPartition error="unknown topic "
  2018-09-05 01:42:06 +0000 [warn]: plugin/output.rb:1157:rescue in try_flush: suppressed same stacktrace
2018-09-05 01:42:09 +0000 [debug]: fluent/log.rb:302:debug: 61 messages send.
2018-09-05 01:42:09 +0000 [warn]: fluent/log.rb:342:warn: Send exception occurred: unknown topic 
2018-09-05 01:42:09 +0000 [warn]: fluent/log.rb:342:warn: Exception Backtrace : /var/lib/gems/2.3.0/gems/ruby-kafka-0.6.8/lib/kafka/protocol/metadata_response.rb:141:in `partitions_for'
/var/lib/gems/2.3.0/gems/ruby-kafka-0.6.8/lib/kafka/cluster.rb:155:in `partitions_for'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:190:in `assign_partitions!'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:153:in `block in deliver_messages_with_retries'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:148:in `loop'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:148:in `deliver_messages_with_retries'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/kafka_producer_ext.rb:102:in `deliver_messages'
/var/lib/gems/2.3.0/gems/fluent-plugin-kafka-0.7.6/lib/fluent/plugin/out_kafka2.rb:220:in `write'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin/output.rb:1110:in `try_flush'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin/output.rb:1389:in `flush_thread_run'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin/output.rb:444:in `block (2 levels) in start'
/var/lib/gems/2.3.0/gems/fluentd-1.2.4/lib/fluent/plugin_helper/thread.rb:78:in `block in thread_create'
2018-09-05 01:42:09 +0000 [info]: fluent/log.rb:322:info: initialized kafka producer: fluentd
2018-09-05 01:42:09 +0000 [debug]: fluent/log.rb:302:debug: taking back chunk for errors. chunk="57515e0ef787da843836cc864f9d1581"
2018-09-05 01:42:09 +0000 [warn]: fluent/log.rb:342:warn: failed to flush the buffer. retry_time=3 next_retry_seconds=2018-09-05 01:42:09 +0000 
```

这是因为没有配置default_topic,使用下面的配置指定topic就可以了。
<!--more-->

```
  <store>
    @type kafka

    brokers 10.12.128.36:9092
    default_topic k8stest
    default_message_key message 

  </store>
```

但是后面还是报错:

```
2018-09-07 10:28:43 +0800 [info]: #0 initialized kafka producer: kafka
2018-09-07 10:28:43 +0800 [warn]: #0 emit transaction failed: error_class=Kafka::ConnectionError error="Could not connect to any of the seed brokers:\n- kafka://10.12.128.36:9092: Connection error EOFError: EOFError" location="/opt/td-a
gent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/cluster.rb:407:in `fetch_cluster_info'" tag="tomcat"
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/cluster.rb:407:in `fetch_cluster_info'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/cluster.rb:367:in `cluster_info'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/cluster.rb:95:in `refresh_metadata!'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/cluster.rb:50:in `add_target_topics'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/producer.rb:276:in `deliver_messages_with_retries'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/producer.rb:238:in `block in deliver_messages'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/instrumenter.rb:23:in `instrument'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/ruby-kafka-0.6.5/lib/kafka/producer.rb:231:in `deliver_messages'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluent-plugin-kafka-0.7.8/lib/fluent/plugin/out_kafka.rb:240:in `emit'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/compat/output.rb:164:in `process'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/output.rb:763:in `emit_sync'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/out_copy.rb:58:in `block in process'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/out_copy.rb:56:in `each'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/out_copy.rb:56:in `each_with_index'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/out_copy.rb:56:in `process'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/multi_output.rb:148:in `emit_sync'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/event_router.rb:96:in `emit_stream'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:385:in `receive_lines'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:503:in `wrap_receive_lines'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:714:in `block in handle_notify'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:758:in `with_io'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:692:in `handle_notify'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:688:in `block in on_notify'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:688:in `synchronize'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:688:in `on_notify'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:534:in `on_notify'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin/in_tail.rb:620:in `on_change'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/cool.io-1.5.3/lib/cool.io/loop.rb:88:in `run_once'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/cool.io-1.5.3/lib/cool.io/loop.rb:88:in `run'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin_helper/event_loop.rb:93:in `block in start'
  2018-09-07 10:28:43 +0800 [warn]: #0 /opt/td-agent/embedded/lib/ruby/gems/2.4.0/gems/fluentd-1.2.2/lib/fluent/plugin_helper/thread.rb:78:in `block in thread_create'
```

后来看网上说是ruby-kafka库的原因，于是就写了一个脚本：

```
require "kafka"

kafka = Kafka.new(["10.12.0.26:9092"], client_id: "my-application")

kafka.deliver_message("Hello, World!", topic: "k8stest")
```

运行后，发现报同样的错误，后再查看ruby-kafka的文档:

Compatibility

|kafka|Producer API| Consumer API|
| ------------ | ------------ | ------------ |
| Kafka 0.8|Full support in v0.4.x|Unsupported|
| Kafka 0.9  |Full support in v0.4.x|Full support in v0.4.x|
|Kafka 0.10|Full support in v0.5.x|Full support in v0.5.x|
|Kafka 0.11|Limited support|Limited support|
|Kafka 1.0|Limited support|Limited support|

这个表也容易给人误解，通常我们会以为高版本的ruby-kafka会支持低版本的kafka，比如居然ruby-kafka 0.4.x支持kafka 0.9，那理所当然，ruby-kafka 0.4.x支持kafka 0.9和0.10吧。但其实这是一一对应的，主要是因为kafka不同版本之间接口的差别过大。
找到原因后就容易了，更换ruby-kafka及fluent-plugin-kafka的版本：

```
td-agent-gem uninstall ruby-kafka --version 0.6.8
td-agent-gem uninstall fluent-plugin-kafka --version 0.7.3
td-agent-gem install ruby-kafka  --version 0.4.3
td-agent-gem install fluent-plugin-kafka  --version 0.6.6
```

最终支持kafka 0.9的ruby-kafka及fluent-plugin-kafka版本为：
```
[root@bs-ops-test-docker-dev-04 ~]# td-agent-gem list | grep kafka
fluent-plugin-kafka (0.6.6)
ruby-kafka (0.4.3)
```
注意：fluent-plugin-kafka 0.7.x的版本不能配合ruby-kafka 0.4.x版本工作。

参考：
https://github.com/zendesk/ruby-kafka
https://blog.csdn.net/mtj66/article/details/79209302