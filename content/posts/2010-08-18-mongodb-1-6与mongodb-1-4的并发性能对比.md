---
title: Mongodb 1.6与Mongodb 1.4的并发性能对比
author: 阿辉
date: 2010-08-18T17:37:00+00:00
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
2010年8月5日，Mongodb 1.6正式发布了，这个版本增加和改进了很多功能，我了解的几个比较大的改进在：

1) Mongodb存储文件申请磁盘空间的方式做了改进。在mongodb  1.4的时候是按128M,256M,512M,1024M,2048M这样的方式申请磁盘空间的；而在mongodb  1.6中，已经是动态小量的申请磁盘空间了。
2) 增加了$or等查询操作符，这在mongodb 1.4的时候是没有的。
3) 改进和提高了并发性能。
4) Replication的同步方面做了改进。
5) etc…

<!--more-->
详细的changelog可以看：
http://jira.mongodb.org/browse/SERVER?report=com.atlassian.jira.plugin.system.project:changelog-panel

在我发表这篇文章时，发现Mongodb 1.6.1也已经发布了，主要是修复了一些bug。

居然说性能得到了提高，那么我们就对Mongodb 1.6和Mongodb 1.4分别做了一个性能测试，想对比下看看Mongodb 1.6性能到底比Mongodb 1.4提高了多少。

测试机器为一台普通台式机，安装在64位centos linux 5.4系统。cpu为Intel E7500，内存为2G,单个普通500G硬盘。

测试程序为自己用java写的，可在此下载：
http://farmerluo.googlecode.com/files/mongotest-0.4.rar

测试程序每次测试都会insert 100万条记录（如10并发测试，每并发insert 10万条记录；20并发测试，每并发5万条记录…），每记录大小为1KB，然后再逐条update所有记录，最后逐条select出来。

测试程序是在本机跑的，所以本次测试忽略网络延时。好，下面我们看测试结果：

下面图中横轴10…100是指并发测试的并发线程。

![/wp-content/uploads/baiduhi/85cad109627b54c1d0581b1f.jpg](/wp-content/uploads/baiduhi/85cad109627b54c1d0581b1f.jpg)

在insert测试内，Mongodb1.4和Mongodb1.6平分秋色，基本上没有区别。虽然在mongodb 1.6中对申请磁盘空间方式做了改进，但对性能的提升没有体现出来。

随着并发的增加，性能快速下降的问题也没有得到改进。

![/wp-content/uploads/baiduhi/8258d1c8422101547e3e6f19.jpg](/wp-content/uploads/baiduhi/8258d1c8422101547e3e6f19.jpg)

在update方面，性能提升显著，合计有75%的性能提高。而且表现平稳，随着并发的增加，性能稳定。非常不错。

![/wp-content/uploads/baiduhi/5ec6092310fd9e0d9358071b.jpg](/wp-content/uploads/baiduhi/5ec6092310fd9e0d9358071b.jpg)

select方面表现也不错，合计有83%性能提高。在并发线程小时尤其明显。

通过上面几个方面测试的结果表明，Mongodb 1.6是还是非常值得我们升级的，不但增加了一些新功能，性能也得到了很大的提高。
