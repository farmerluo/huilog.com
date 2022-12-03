---
title: 一个配置heartbeat 2.x的小技巧
author: 阿辉
date: 2008-11-03T22:17:00+00:00
categories:
- Heartbeat
tags:
- Heartbeat
keywords:
- Heartbeat
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
2.x开始使用cib.xml作为配置文件，当heartbeat启用的时候，无法手工修改cib.xml，即使修改保存了也无效，因为会被cib.xml.last覆盖掉，这个时候您需要停掉heartbeat，删除`/var/lib/heartbeat/crm`中的`cib.xml.sig cib.xml.last cib.xml.last.sig`，然后修改保存，再启动heartbeat，单单启动heartbeat就要花费5分钟（如果你的ha.cf的配置是默认的话），所以排错的时候非常麻烦。
<!--more-->
昨天我在ha的mail-list中看到一则小技巧，在heartbeat运行的时候，可以将cib倒出来，然后修改，再导回去，可以立即生效！

```
cibadmin -Q > tmp.xml

vim tmp.xml

cibadmin -R -x tmp.xml
```

非常方便！