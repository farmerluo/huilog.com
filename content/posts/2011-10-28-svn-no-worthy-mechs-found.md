---
title: 'svn: No worthy mechs found'
author: 阿辉
date: 2011-10-28T17:27:00+00:00
categories:
- SVN
tags:
- SVN
keywords:
- SVN
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
今天发现使用svn up之后，/var/log/messages内有个错误：

Oct 28 13:36:48 web1 svn: No worthy mechs found  
Oct 28 13:58:14 web1 svn: No worthy mechs found  
Oct 28 14:07:18 web1 svn: No worthy mechs found

网上查了下，是因由于加密认证的库没有装上导致的，需要安装cyrus-sasl-md5：
```bash
yum -y install cyrus-sasl-md5
```

<!--more-->

用上面的命令安装cyrus-sasl-md5后，再svn up就没报错了。

不过svn: No worthy mechs found的问题，并不影响使用。
