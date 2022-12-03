---
title: Squid中的日志出现TCP_CLIENT_REFRESH_MISS的问题排除
author: 阿辉
date: 2008-10-29T21:14:00+00:00
categories:
- Squid
tags:
- Squid
keywords:
- Squid
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
今天检查Squid发现大量的日志出现TCP_CLIENT_REFRESH_MISS,见到Cacti中的流量，自己的CDN节点命令不高,查了很久的原因，最后发现,原来是exe的文件下载的原因.

花了半天的时间才解决这个问题，要解决这个问题。

range_offset_limit 和 reload_into_ims
<!--more-->
range_offset_limit这个参数，主要是对各种流媒体和要断点续传的文件的缓存的。缺省是0，也就是说 只要client发过来的http header里包含了“Range:” ，squid会把这个请求转到后端http server，最致命的是，http server返回的内容并不会进入squid的cache store。

range_offset_limit就派上用场了，把它的值设置为-1；然后squid会把Range头去掉，而从后端http server把内容全部抓下来，放到cache store里，随后的对该cache的访问就不再转发到后端http server，可以大大提高命中率。也可以给这个参数设置一个值,会提前下载多少内容.但要注意，这个参数不要大过 maximum_object_size ，不然下载完了,maximum_object_size 这个参数不能缓存这么多,又删除这个文件.白白点用你的流量.

reload_into_ims 这个参数，要分清哦，不是refresh_pattern中的参数，他是一个单独的参数.
在flashget和迅雷之类的软件，下载时发送http header时会包含下列信息：
```
Cache-Control:no-cache
Pragma:no-cache
```
这样的话squid主机接受这http header以后会让squid服务器直接连接web server取新的数据。这样对服务器很大的压力，因为服务器的过期时间是后面的程度控制，不方便用refresh_pattern的 ignore-reload 参数强行忽略请求里的任何no-cache指令,这时只有使用reload_into_ims这个参数.

打开这个参数为on ,就行,这个参数违反 HTTP 协议,但是对大部分网站来说是可以设置为 on 的，只要后端服务器对
If-Modified-Since 头的判断正确即可。

下面是参数解释

When you enable this option, client no-cache or ``reload'' requests will be changed to If-Modified-Since requests.

如果客户端送过来no-cache的http头，和刷新重新载入他的浏览器时,装改变成请求变成 If-Modified-Since的请求.