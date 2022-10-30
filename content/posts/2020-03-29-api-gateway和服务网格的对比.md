---
title: Api Gateway和服务网格的对比
author: 阿辉
date: 2020-03-28T20:10:08+00:00
categories:
- Api Gateway
tags:
- Api Gateway
keywords:
- Api Gateway
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

在服务网络出来之前，就已经有很多各式各样的Api Gateway了。他们之间有什么不同呢？

# 1. Api Gateway架构
Api Gateway有很多，现阶段做的比较好的有ambassador和kong。还有一些其它的，目前看来这两个用的人比较多。
- Ambassador架构：
![](http://www.huilog.com/wp-content/uploads/2020/03/9e3267fccecd244e9778f39c2892ec0c.png)

- Kong架构：
![](http://www.huilog.com/wp-content/uploads/2020/03/f58127d7d80a5383fb57a1fca11b11f3.png)

从流量转发的角度来看，Api Gateway主要做的事是外部到内部的流量转发，也就是通常所说的南北流量。基于kubernetes的架构内来说，其主要是实现ingress的功能。

应用内部之间的流量（东西流量）并不是它关注的重点，虽然部分上也可以实现，但是相关功能并不多。

# 2. API Gateway解决了什么问题？
API Gateway主要是解决了API管理的问题。

根据Wikipedia的定义：API管理的过程包括创建和发布Web API、执行调用策略、访问控制、订阅管理、收集和分析调用统计以及报告性能。

根据以上定义，API Gateway主要的功能如下：
- api的创建和发布，比如说api的自动注册
- 访问控制和调用策略（流控等），谁能访问？API被调用时，怎么确保安全和稳定的访问？
- 订阅管理，哪些人可以访问我的API
- 调用性能分析及信息收集等

<!--more-->

# 3. 服务网络（Service Mesh）架构
![](http://www.huilog.com/wp-content/uploads/2020/03/03a5f36bf121dcd95cde94743a592292.png)

![](http://www.huilog.com/wp-content/uploads/2020/03/585286619015acdadceb71ff83652a65.png)

在微服务一统天下的时代，微服务数量成百上千，一个API通常要经过几个或几十个微服务，微服务之间的管理越来越复杂，这就是服务网络出现的原因。

可见服务网络的侧重点在解决微服务之间服务调用的问题。也就是所谓的东西流量的问题。

# 4. 服务网络（Service Mesh）解决了什么问题？
服务网络解决了什么问题呢？主要有如下：
- 弹性：超时、重试、熔断、故障处理、负载均衡。
- 流量限制：基于多个来源的基础设施流量限制。
- 通信路由：根据path、host、header、cookie base、源Service……
- 可观测性：指标、日志、分布式追踪。
- 安全：mTLS、RBAC……

# 5. 简单总结

经过上面的对比，从运维的角度来看的话。简单的类比就是：
- Api Gateway实际上相当于外网网关层。
- 服务网络相当于内网网关层（当然它把外网网关层也做了，只是侧重点在内网网关层）。

