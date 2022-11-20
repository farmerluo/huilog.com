---
title: Intel X25-E 64G固态硬盘做raid5与SAS 300G硬盘做raid10性能对比
author: 阿辉
date: 2010-04-15T10:19:00+00:00
categories:
- 硬件
tags:
- 硬件
keywords:
- 硬件
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
最近公司弄了一台dell pe r710来做数据库，里面配了4块Intel® X25-E 64G的固态硬盘，正好还有另外两台数据库服务器也是dell pe r710的，配置基本一样，只是硬盘配的是SAS 300G，15K转的SAS硬盘。我正好趁这个机会，对两台服务器的磁盘做了一次测试：

Intel X25-E 64G X 4 做 Raid 5

VS

SAS 300G X 4 做Raid 10
<!--more-->

下面图中190GB的是固态硬盘，598GB的是SAS硬盘，从图中可以看到固态硬盘文件读要比SAS硬盘高出近一倍，文件写也高出不少。

对数据库来说最重要的是随机读写中的IOPS，从图中可以看到，固态硬盘的IOPS要高出SAS硬盘10到20倍。可见用固态硬盘来跑数据库是再合适不过了。可惜的是目前固态硬盘容量还是有点小，大的太贵，为了增加一点容量，我们也只能做raid5用了。

啥也不说了，上图：

![/wp-content/uploads/baiduhi/b014c9179a05ad3ec83d6d61.jpg](/wp-content/uploads/baiduhi/b014c9179a05ad3ec83d6d61.jpg)
![/wp-content/uploads/baiduhi/0e4abc3e660ef9cd838b1361.jpg](/wp-content/uploads/baiduhi/0e4abc3e660ef9cd838b1361.jpg)
![/wp-content/uploads/baiduhi/233a5edf4a8a012448540361.jpg](/wp-content/uploads/baiduhi/233a5edf4a8a012448540361.jpg)
![/wp-content/uploads/baiduhi/877d76c612d4042c9c163d61.jpg](/wp-content/uploads/baiduhi/877d76c612d4042c9c163d61.jpg)
![/wp-content/uploads/baiduhi/b45e7cd92e2bcbde38012f61.jpg](/wp-content/uploads/baiduhi/b45e7cd92e2bcbde38012f61.jpg)
![/wp-content/uploads/baiduhi/b6d6ad34ceb97880d1a2d361.jpg](/wp-content/uploads/baiduhi/b6d6ad34ceb97880d1a2d361.jpg)
