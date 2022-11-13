---
title: yum 报错：pkgKey 1445 doesn’t exist in repo base
author: 阿辉
date: 2013-01-30T02:11:46+00:00
categories:
- Linux
tags:
- Linux
keywords:
- Linux
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
执行`yum -y update`报错：`pkgKey 1445 doesn't exist in repo base.`

系统提示进行如下操作：
```
package-cleanup --problems
package-cleanup --dupes
rpm -Va --nofiles --nodigest
```
<!--more-->

但其实没有作用，真正有用的是下面两个命令：
```
yum clean all
yum clean metadata 
```
把缓存清理一下，然后再：
```
yum -y update
```
就好了。