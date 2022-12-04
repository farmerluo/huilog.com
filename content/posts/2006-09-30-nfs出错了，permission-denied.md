---
title: NFS出错了，Permission denied
author: 阿辉
date: 2006-09-30T11:31:00+00:00
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
今天同事跟我说网站的图片不能显示了，后来检查的时候发现一个ＮＦＳ的怪异现象。

在mount nfs目录时出现错误：
```bash
mount -t nfs 192.168.1.172:/nfs/mp3/mp3files /web/mp3/mp3files
mount: 192.168.1.172:/nfs/mp3/mp3files failed, reason given by server: Permission denied
```

我原来的/etc/exportfs是这样的：
```bash
[root@ha1 nfs]# cat /etc/exports
/nfs/mp3/mp3files 192.168.1.*(rw,async)
```

一直都用的好好的，其它的机器通过内网ＩＰ来mount这台上面的数据。

所以我想应该是我做了什么造成的，因为之前我看到/var/log/messages:

`mountd[3082]: Fake hostname rs0.xxxxxxcom for 192.168.1.69 - forward lookup doesn’t exist`

<!--more-->

以为nfs警告说我没有做域名反解，所以我就在我的域名服务器做把192.168.1.69做了一下反解。并增加了rs0.xxxxx.com这个域名到192.168.1.69。做完之后就没有再出现上面的错误了，但是上面说的出现不能mount的情况。

在网上查了一些资料，有人说把/etc/exports换成域名试试，所以我就改成了：
```bash
[root@ha1 nfs]# cat /etc/exports
/nfs/mp3/mp3files *.xxxxxx.com(rw,async)
```
再mount，发现正常，没有问题了。

后来又查了一些相关资料，才知道：

nfs server接到客户端的mount时，会先客户的IP做反解成域名，用域名(注意是用域名而不是ＩＰ)去和/etc/exports做比较，如果匹配不成功会失败。

而我做了域名反解后，并没有更新/etc/exports内的ＩＰ为域名。所以匹配不到对应的域名，自然就出现`mount: 192.168.1.172:/nfs/mp3/mp3files failed, reason given by server: Permission denied的`错误了。

之前用ＩＰ没有问题是因为在域名不能反解的时候还是用ＩＰ去匹配的。