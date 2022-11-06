---
title: nginx使用map配置AB测试环境
author: 阿辉
date: 2018-01-16T10:15:33+00:00
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
通过使用nginx的map可以配置按照不同的值（如URL参数，POST值,cooke，http头等等...）,来转到不用的后端主机，以实现简单的AB测试环境。

下面为一个配置的例子：
```bash
# underscores_in_headers on;
#当使用超过一个map时，下面两个参数需要调大，否则会报错
map_hash_max_size 262144;
map_hash_bucket_size 262144;

#定义3个upstream用于AB测试，用于显示不同的内容，相应下面有建3个虚拟主机端口与此对应
upstream def {
        server 127.0.0.1:8080;
}

upstream a {
        server 127.0.0.1:8081;
}

upstream b {
        server 127.0.0.1:8082;
}

# 定义map，$cookie_aid是指从cookie内取aid的值,当cookie内aid的值为"12345"时，$upserver为a，$upserver我这边是用的是upstrem的名称，其它类似
map $cookie_aid $upserver {
        default                          "def";
        "12345"                          "a";
        "67890"                          "b";
}
# 从$http_pid内取值,是从http header内取pid的的值,当http header内pid的值为"12345"时，$headerpid 为a，其它类似
map $http_pid $headerpid {
        default                          "def";
        "12345"                          "a";
        "67890"                          "b";
}
# 从$host内取值,$host对应的是用户访问的域名,当域名的值为12345.abtest-b.xxxxx.com时，$bserver 为a，其它类似
map $host $bserver {
		12345.abtest-b.xxxxx.com       "a";
		67890.abtest-b.xxxxx.com       "b";
		.abtest-b.xxxxx.com            "def";
		default                        "def";
}
# 从$arg_userid内取值,$arg_userid对应的是用户访问的URL上所带的参数userid的值,当userid的值为12345时，$userid为a，其它类似
map $arg_userid $userid {
        default                          "def";
        "12345"                          "a";
        "67890"                          "b";
}

server {
        listen 80;
        server_name abtest.xxxxx.com;
        charset utf-8;
        root /data/www/a/;
        access_log /data/log/nginx/abtest.internal.weimobdev.com_access.log main;
        error_log /data/log/nginx/abtest.internal.weimobdev.com_error.log info;

        location / {
                # 通过以下的if语句,实现多个变量的判断,当cookie内取aid的值判断完后，再判断http header内pid的值
                if ( $upserver = "def" ) {
                         set $upserver  $headerpid;
                }
                # 由于$upserver我这边是用的是upstrem的名称,所以可以直接转发
                proxy_pass http://$upserver;   
        }
}

server {
        listen 80;
        server_name abtest-b.xxxxx.com *.abtest-b.xxxxx.com;
        charset utf-8;
        root /data/www/a/;
        access_log /data/log/nginx/abtest-b.xxxxx.com_access.log main;
        error_log /data/log/nginx/abtest-b.xxxxx.com_error.log info;

        location / {
                # 通过以下的if语句,实现多个变量的判断,当$host内的域名判断完后，再判断URL上所带的参数userid的值
                if ( $bserver = "def" ) {
                         set $bserver $userid;
                }
                # 
                proxy_pass http://$bserver;
        }
}

server {
        listen 8080;
        charset utf-8;
        root /data/www/default/;
        access_log /data/log/nginx/default.log;
}

server {
        listen 8081;
        charset utf-8;
        root /data/www/a/;
        access_log /data/log/nginx/a.log;
}

server {
        listen 8082;
        charset utf-8;
        root /data/www/b/;
        access_log /data/log/nginx/b.log;
}
```
<!--more-->

配置好重载nginx配置，可以使用curl做测试, -b "aid=12345"为发送cookie aid，-H "pid:12345"为发送http header。

以下是nginx的内置参数：

nginx的配置文件中可以使用的内置变量以美元符$开始，也有人叫全局变量。其中，部分预定义的变量的值是可以改变的。

$arg_PARAMETER 这个变量值为：GET请求中变量名PARAMETER参数的值。

$args 这个变量等于GET请求中的参数。例如，foo=123&bar=blahblah;这个变量只可以被修改

$binary_remote_addr 二进制码形式的客户端地址。

$body_bytes_sent 传送页面的字节数

$content_length 请求头中的Content-length字段。

$content_type 请求头中的Content-Type字段。

$cookie_COOKIE cookie COOKIE的值。

$document_root 当前请求在root指令中指定的值。

$document_uri 与$uri相同。

$host 请求中的主机头(Host)字段，如果请求中的主机头不可用或者空，则为处理请求的server名称(处理请求的server的server_name指令的值)。值为小写，不包含端口。

$hostname 机器名使用 gethostname系统调用的值

$http_HEADER HTTP请求头中的内容，HEADER为HTTP请求中的内容转为小写，-变为_(破折号变为下划线)，例如：$http_user_agent(Uaer-Agent的值), $http_referer...;

$sent_http_HEADER HTTP响应头中的内容，HEADER为HTTP响应中的内容转为小写，-变为_(破折号变为下划线)，例如： $sent_http_cache_control, $sent_http_content_type...;

$is_args 如果$args设置，值为"?"，否则为""。

$limit_rate 这个变量可以限制连接速率。

$nginx_version 当前运行的nginx版本号。

$query_string 与$args相同。

$remote_addr 客户端的IP地址。

$remote_port 客户端的端口。

$remote_user 已经经过Auth Basic Module验证的用户名。

$request_filename 当前连接请求的文件路径，由root或alias指令与URI请求生成。

$request_body 这个变量（0.7.58+）包含请求的主要信息。在使用proxy_pass或fastcgi_pass指令的location中比较有意义。

$request_body_file 客户端请求主体信息的临时文件名。

$request_completion 如果请求成功，设为"OK"；如果请求未完成或者不是一系列请求中最后一部分则设为空。

$request_method 这个变量是客户端请求的动作，通常为GET或POST。
包括0.8.20及之前的版本中，这个变量总为main request中的动作，如果当前请求是一个子请求，并不使用这个当前请求的动作。

$request_uri 这个变量等于包含一些客户端请求参数的原始URI，它无法修改，请查看$uri更改或重写URI。

$scheme 所用的协议，比如http或者是https，比如rewrite ^(.+)$ $scheme://example.com$1 redirect;

$server_addr 服务器地址，在完成一次系统调用后可以确定这个值，如果要绕开系统调用，则必须在listen中指定地址并且使用bind参数。

$server_name 服务器名称。

$server_port 请求到达服务器的端口号。

$server_protocol 请求使用的协议，通常是HTTP/1.0或HTTP/1.1。

$uri 请求中的当前URI(不带请求参数，参数位于$args)，不同于浏览器传递的$request_uri的值，它可以通过内部重定向，或者使用index指令进行修改。不包括协议和主机名，例如/foo/bar.html

使用curl 指定域名IP测试的例子：

方法一

`curl url -x ip:port`

例`curl http://baidu.com -x 10.12.20.21:80`

这个方法访问是正常，不过访问日志中$request字段显示的是”GET http://baidu.com”

方法二

`curl -H ‘Host:baidu.com’ http://10.12.20.21`

这个方法显示是正常的，日志也正常，推荐这种方法

更多例子：
```
-b/--cookie <name=string/file> Cookie string or file to read cookies from (H)
-d/ --data <data> HTTP POST data (H)
--data-ascii <data> HTTP POST ASCII data (H)
--data-binary <data> HTTP POST binary data (H)
--data-urlencode <name=data/name@filename> HTTP POST data url encoded (H)
--delegation STRING GSS-API delegation permission
--digest Use HTTP Digest Authentication (H)
--disable-eprt Inhibit using EPRT or LPRT (F)
--disable-epsv Inhibit using EPSV (F)
-H/--header <line> Custom header to pass to server (H)
-I/--head Show document info only

curl http://1234.m.xxx.com/o2o/ -x 127.0.0.1:80

curl -b "aid=55875671" -H 'Host:o2oopen.xxx.com' http://127.0.0.1/

curl -v -H 'pid:55875671' -b "aid=55875671" -x http://127.0.0.1:80 http://o2oopen.xxx.com/

curl -v -H 'pid:1234' -b "aid=55875671" -x http://127.0.0.1:80 http://o2oopen.xxx.com/
```