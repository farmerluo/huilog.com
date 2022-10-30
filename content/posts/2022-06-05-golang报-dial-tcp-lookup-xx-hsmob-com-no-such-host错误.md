---
title: golang报 dial tcp lookup xx.hsmob.com no such host错误
author: 阿辉
date: 2022-06-05T09:55:45+00:00
categories:
- Coding
- Golang
tags:
- Coding
- Golang
keywords:
- Coding
- Golang
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
在我的mac上，运行golang程序调用http接口报如下错误：  
golang报 dial tcp lookup xx.hsmob.com no such host错误

但实际域名是通的，报错如下:

```bash
W0527 10:54:34.046009   16869 zeus.go:141] not found members in appid:xmn-sys-comm-setting-bloc-hotel, error: Get "http://cmdbv3.internal.hsmob.com/api/v3/application/xmn-sys-comm-setting-bloc-hotel/member": dial tcp: lookup cmdbv3.internal.hsmob.com: no such host
W0527 10:54:34.046236   16869 zeus.go:141] not found members in appid:xmn-sys-comm-setting-flat, error: Get "http://cmdbv3.internal.hsmob.com/api/v3/application/xmn-sys-comm-setting-flat/member": dial tcp: lookup cmdbv3.internal.hsmob.com: no such host
W0527 10:54:34.046475   16869 zeus.go:141] not found members in appid:saas-fe-login-frontend, error: Get "http://cmdbv3.internal.hsmob.com/api/v3/application/saas-fe-login-frontend/member": dial tcp: lookup cmdbv3.internal.hsmob.com: no such host
```

<!--more-->

后来网上查了下，说可能是file descriptors的问题。

```bash
luohui@luohuideMBP16 ~/yaml $ ulimit -a
-t: cpu time (seconds)              unlimited
-f: file size (blocks)              unlimited
-d: data seg size (kbytes)          unlimited
-s: stack size (kbytes)             8176
-c: core file size (blocks)         0
-v: address space (kbytes)          unlimited
-l: locked-in-memory size (kbytes)  unlimited
-u: processes                       5333
-n: file descriptors                256
luohui@luohuideMBP16 ~/yaml $ ulimit -n 655350
```

调大就好了。

```bash
umlimit -n 81920
```