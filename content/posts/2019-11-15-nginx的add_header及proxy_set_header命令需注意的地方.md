---
title: nginx的add_header及proxy_set_header命令需注意的地方
author: 阿辉
date: 2019-11-15T08:25:13+00:00
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
nginx的add_header及proxy_set_header命令有三个地方可以配置，分别是http,server,location。我们一般对其优先级的理解是location->server->http。

这没有错，但是这个优先级是指令级别的（不包括指令后面的参数），也就是说：假设在http和server内都有add_header,那么只有server内的生效，忽略http内的add_header，而不管在这两个地方配置新增的header是不是一样。

proxy_set_header也是一样的效果。所以个人推测有很多nginx指令可能都是这样的规则。

另个在同一级别也有坑，对于add_header，如果同一个域名的两个不同location加了不同的add_header，那么只有最后一个location内的生效。这个我没有亲测，详细的可以看下面的参考链接。

<!--more-->

> 官网add_header的文档，有这样的描述（其他信息已省略）：

>    There could be several add_header directives. These directives are inherited from the previous level if and only if there are no add_header directives defined on the current level.

> 注意重点在“These directives are inherited from the previous level if and only if there are no add_header directives defined on the current level. ”。即：仅当当前层级中没有add_header指令才会继承父级设置。所以我的疑问就清晰了：location中有add_header，nginx.conf中的配置被丢弃了。

> 这是Nginx的故意行为，说不上是bug或坑。但深入体会这句话，会发现更有意思的现象：仅最近一处的add_header起作用。http、server和location三处均可配置add_header，但起作用的是最接近的配置，往上的配置都会失效。

> 但问题还不仅于此。如果location中rewrite到另一个location，最后结果仅出现第二个的header。例如：

```
location /foo1 {
    add_header foo1 1;
    rewrite / /foo2;
}

location /foo2 {
    add_header foo2 1;
    return 200 "OK";
}
```
> 不管请求/foo1还是/foo2，最终header只有foo2：

> 尽管说得通这是正常行为，但总让人感觉有点勉强和不舒坦：server丢掉http配置，location丢掉server配置也就算了，但两个location在同一层级啊！

> 不能继承父级配置，又不想在当前块重复指令，解决办法可以用include指令。

参考：
https://tlanyan.me/be-careful-with-nginx-add_header-directive/