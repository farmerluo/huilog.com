---
title: Mysql及MongoDB并发测试对比
author: 阿辉
date: 2010-04-23T16:08:00+00:00
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
我在前面的文章内说过，准备做一个MongoDB与Mysql的并发对比测试，经过近一周的测试，已经完成，结果如下：

![/wp-content/uploads/baiduhi/9519a61e8f228327403417df.jpg](/wp-content/uploads/baiduhi/9519a61e8f228327403417df.jpg)

<!--more-->

MongoDB与Mysql的并发对比测试曲线图：

![/wp-content/uploads/baiduhi/ae80c8fcfc3fbdc5fd037fdf.jpg](/wp-content/uploads/baiduhi/ae80c8fcfc3fbdc5fd037fdf.jpg)

我们大致总结一下：

通过数据和图表可以看到，在并发测试下，MongoDB对Mysql的优势没有在单用户那么大.了（可参考本空间前面的测试文章）。Mongodb的insert及update性能大约是Mysql的2~3倍，select方面MongoDB与Mysql相当。

测试环境：

CPU：Intel(R) Core(TM)2 Duo CPU     E7500  @ 2.93GHz

内存：4G

硬盘：普通SATA硬盘 500G

OS  ：Centos Linux 5.4 X64

Mysql版本为：5.0.77 ，MongoDB为自己编译的1.40。

以上结果是由之前写的Java程序（http://farmerluo.googlecode.com/files/mongotest-0.4.rar）测试出来的，因为只有一台测试机器，测试程序是和mysql及MongoDB在同一台机器上运行测试的。