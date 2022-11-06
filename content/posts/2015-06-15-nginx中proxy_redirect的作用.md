---
title: NGINX中proxy_redirect的作用
author: 阿辉
date: 2015-06-15T06:22:40+00:00
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
NGINX的proxy_redirect功能比较强大,其作用是对发送给客户端的URL进行修改。以例子说明：

例一：
```
   server {
       listen       80;
       server_name  test.abc.com;
       location / {
            proxy_pass http://10.10.10.1:9080;
       }
   }
```

这段配置一般情况下都正常,但偶尔会出错, 错误在什么地方呢?

抓包发现服务器给客户端的跳转指令里加了端口号,如 `Location: http://test.abc.com:9080/abc.html` 。因为nginx服务器侦听的是80端口，所以这样的URL给了客户端,必然会出错.针对这种情况, 加一条proxy_redirect指令: `proxy_redirect http://test.abc.com:9080/ / `,把所有`http://test.abc.com:9080/`的内容替换成`/`再发给客户端，就解决了。
```
   server {
       listen       80;
       server_name  test.abc.com;
       proxy_redirect http://test.abc.com:9080/ /;
       location / {
            proxy_pass http://10.10.10.1:9080;
       }
   }
```
<!--more-->

例二：

下面这段环境是这样，浏览器访问`https://account.abc.com/`会转至后端的`http://proxy/account/`上。是通过rewrite做的。
```
   server {
        listen       443;
        server_name  account.abc.com;

        ssl                         on;
        ssl_certificate             abc.com.pem;
        ssl_certificate_key         abc.com.key;
        ssl_session_timeout         5m;
        ssl_protocols               TLSv1.2 TLSv1.1 TLSv1;
        ssl_ciphers                 HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers   on;

        root   /data1/www/;
        index  index.php index.html index.htm;

        access_log /data1/log/nginx/account.abc.com-https.log main;
        error_log  /data1/log/nginx/account.abc.com-https-error.log warn;

        rewrite ^/(.*)$ /account/$1 last;

        location /account {
            proxy_pass  http://account;
            proxy_cookie_path /account/ /;
        }

   }
```

没有特殊情况，这么做一直是OK的。

但是当后端的服务器需要做一个302跳转时，问题就出来了。会跳转到类似的URL：
`http://account.abc.com/account/xxxx`

而我们希望的跳转URL其实是：

`https://account.abc.com/xxxx`

解决办法是使用proxy_redirect改写返回到浏览器的URL：

```
   server {
        listen       443;
        server_name  account.abc.com;

        ssl                         on;
        ssl_certificate             abc.com.pem;
        ssl_certificate_key         abc.com.key;
        ssl_session_timeout         5m;
        ssl_protocols               TLSv1.2 TLSv1.1 TLSv1;
        ssl_ciphers                 HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers   on;

        root   /data1/www/;
        index  index.php index.html index.htm;

        access_log /data1/log/nginx/account.abc.com-https.log main;
        error_log  /data1/log/nginx/account.abc.com-https-error.log warn;

        rewrite ^/(.*)$ /account/$1 last;

        location /account {
            proxy_pass  http://account;
            proxy_cookie_path /account/ /;
            proxy_redirect http://account.abc.com/account $scheme://account.abc.com;
        }

   }
```

### proxy_redirect官方说明

#### 语法：proxy_redirect [ default|off|redirect replacement ]  
默认值：proxy_redirect default  
使用字段：http, server, location  
如果需要修改从被代理服务器传来的应答头中的&#8221;Location&#8221;和&#8221;Refresh&#8221;字段，可以用这个指令设置。  
假设被代理服务器返回Location字段为： `http://localhost:8000/two/some/uri/`
这个指令：

`proxy_redirect http://localhost:8000/two/ http://frontend/one/;`

#### 将Location字段重写为http://frontend/one/some/uri/。  
在代替的字段中可以不写服务器名：

`proxy_redirect http://localhost:8000/two/ /;`

#### 这样就使用服务器的基本名称和端口，即使它来自非80端口。  
如果使用“default”参数，将根据location和proxy_pass参数的设置来决定。  
例如下列两个配置等效：

```
location /one/ {
  proxy_pass       http://upstream:port/two/;
  proxy_redirect   default;
}
 
location /one/ {
  proxy_pass       http://upstream:port/two/;
  proxy_redirect   http://upstream:port/two/   /one/;
}
```

#### 在指令中可以使用一些变量：

`proxy_redirect   http://localhost:8000/    http://$host:$server_port/;`

#### 这个指令有时可以重复：

```
  proxy_redirect   default;
  proxy_redirect   http://localhost:8000/    /;
  proxy_redirect   http://www.example.com/   /;
```

#### 参数off将在这个字段中禁止所有的proxy_redirect指令：

```
  proxy_redirect   off;
  proxy_redirect   default;
  proxy_redirect   http://localhost:8000/    /;
  proxy_redirect   http://www.example.com/   /;
```

#### 利用这个指令可以为被代理服务器发出的相对重定向增加主机名：

`proxy_redirect   /   /;`
