---
title: nginx rewrite 的一些参数
author: 阿辉
date: 2008-11-24T15:08:00+00:00
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
正则表达式匹配，其中：
```
* ~ 为区分大小写匹配
* ~* 为不区分大小写匹配
* !~和!~*分别为区分大小写不匹配及不区分大小写不匹配
```

文件及目录匹配，其中：
```
* -f和!-f用来判断是否存在文件
* -d和!-d用来判断是否存在目录
* -e和!-e用来判断是否存在文件或目录
* -x和!-x用来判断文件是否可执行
```

flag标记有：
```
* last 相当于Apache里的[L]标记，表示完成rewrite
* break 终止匹配, 不再匹配后面的规则
* redirect 返回302临时重定向
* permanent 返回301永久重定向
```

一些可用的全局变量有，可以用做条件判断(待补全)
```
$args
$content_length
$content_type
$document_root
$document_uri
$host
$http_user_agent
$http_cookie
$limit_rate
$request_body_file
$request_method
$remote_addr
$remote_port
$remote_user
$request_filename
$request_uri
$query_string
$scheme
$server_protocol
$server_addr
$server_name
$server_port
$uri
```

举例:
```
abc.domian.com/sort/2 => abc.domian.com/index.php?act=sort&name=abc&id=2
if ($host ~* (.*).domain.com) {
    set $sub_name $1;
    rewrite ^/sort/(d+)/?$ /index.php?act=sort&cid=$sub_name&id=$1 last;
}
```

http://wiki.codemongers.com/NginxChsHttpRewriteModule
http://www.romej.com/archives/515/nginx-rewrite-rules-for-wordpress-redux
http://info.codepub.com/2008/08/info-21590.html