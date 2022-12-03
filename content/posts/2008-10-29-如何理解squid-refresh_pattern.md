---
title: 如何理解Squid refresh_pattern
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
refresh_pattern的作用:
用于确定一个页面进入cache后，它在cache中停留的时间。

语法：

`refresh_pattern [-i] regexp min percent max [options]`

几个概念:

resource age =对象进入cache的时间-对象的last_modified

response age   =当前时间-对象进入cache的时间

LM-factor=(response age)/(resource age)

<!--more-->
举个例子,这里只考虑percent, 不考虑min   和 max

例如：refresh_pattern 20%

假设源服务器上`www.aaa.com/index.htm   —–lastmodified` 是2007-04-10 02:00:00

squid上    `proxy.aaa.com/index.htm   index.htm`进入cache的时间2007-04-10 03:00:00

1）如果当前时间 2007-04-10 03:00:00
```
resource age =3点-2点=60分钟
response age =0分钟
index.htm还可以在cache停留的时间(resource age)20%=12分钟
```
也就是说，index.htm进入cache后，可以停留12分钟，才被重新确认。


2）如果当前时间   2007-04-10 03:05:00
```
resource age =3点-2点=60分钟
response age =5分钟
index.htm还可以在cache停留的时间(resource age)20%=12分钟-5=7
LM-factor=5/60=8.3%<20%
```
一直到2007-04-10 03:12:00 LM-factor=12/60=20% 之后，cache中的页面index.htm终于stale。

如果这时没有index.htm的请求，index.htm会一直在缓存中，如果有index.htm请求，squid收到该请求后，由于已经过 期，squid会向源服务器发一个index.htm是否有改变的请求，源服务器收到后，如果index.htm没有更新，squid就不用更新缓存，直 接把缓存的内容放回给客户端，同时，重置对象进入cache的时间为与源服务器确认的时间，比如2007-04-10 03:13:00，如果正好在这个后重新确认了页面。重置后，resource age变长，相应在cache中存活的时间也变长。

如果有改变则把最新的index.htm返回给squid,squid收到会更新缓存，然后把新的index.htm返回给客户端,同时根据新页面中的Last_Modified和取页面的时间，重新计算resource age，进一步计算出存活时间。

实际上，一个页面进入cache后，他的存活时间就确定了，即 (resource age) * 百分比，一直到被重新确认。

理解了百分比后,min max就好理解了

squid收到一个页面请求时：
1、计算出response age，
2、如果`response age < min` 则 fresh   如果`response age>max` 则 stale
3、如果response age在之间，如果response时间<存活时间，fresh，否则stale