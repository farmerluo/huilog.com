---
title: load balancer后端nginx封IP的方法
author: 阿辉
date: 2012-12-24T08:16:50+00:00
categories:
- Nginx
tags:
- Nginx
keywords:
- Nginx
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
周末发现在个IP恶意访问，我们就想先封掉这个IP地址。

我们的环境是前端有个load balancer,后端有几台WEB服务器。load balancer上不好封IP地址，那就只能配置在后端WEB服务器的nginx上了。

很简单，我很快就配置上去了，如下面这样的：

```
deny 82.245.163.1;
allow all
```

<!--more-->

发现不起作用，仔细想了一下，HttpAccessModule应该是取客户端真实IP的，而在后端WEB服务器上，客户端上的真实IP是在头部的X-Forwarded-For内。

然后就改成如下的方式封IP：

```
set $allow true;

if ($http_x_forwarded_for ~ " ?82.245.163.1$") {
    set $allow false ;
}

if ($allow = false) {
    return 403;
}
```

测试一下，发现可以了。