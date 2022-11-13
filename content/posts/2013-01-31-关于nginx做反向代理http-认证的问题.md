---
title: 关于nginx做反向代理http 认证的问题
author: 阿辉
date: 2013-01-31T06:45:55+00:00
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
使用nginx做反向代理，当后端服务器需要认证时，需要把认证的http header也传到后端服务器上去，配置为：
```
proxy_set_header Authorization $http_authorization;
```
这样配置后，服务器会发出一个认证窗口出来，提示用户输入用户名密码。

如果不想让用户输入用户名密码，可用下面的配置：
```
proxy_hide_header WWW-Authenticate; //隐藏发给用户的认证http header，相当于不提示用户输用户名密码了。
proxy_set_header Authorization "Basic dXNlcjpwYXNzd29yZA==";  //发送httpd 认证 header给后端服务器。
```
`dXNlcjpwYXNzd29yZA==`是`base64(user:pass)`得到的。

解释一下上面两个http header：

WWW-Authenticate： 这是GET的时候带的，服务器发给客户端的。表明客户端请求实体应该使用的授权方案  

示例：`WWW-Authenticate: Basic`

Basic是基本的http认证方式，除此之外还有NTLM等，NTLM是微软的，nginx目前不支持。

Authorization     这个是用户输入用户名和密码后，POST到服务器的时候带的。HTTP授权的授权证书 

示例：`Authorization: Basic QWxhZGRpbjpvcGVuIHNlc2FtZQ==`

<!--more-->

下面是一个提示用户输入用户名和密码的httpd auth反向代理的配置例子：
```
   upstream dwserver {          
      server 10.18.0.1:81 max_fails=3 fail_timeout=30s;
   }



   server {
        listen       81;
        server_name  dwout.test.com;

        location ~ .*$ {

            proxy_redirect off;
#            proxy_pass_request_headers on;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#            proxy_hide_header Authorization;
#            proxy_hide_header WWW-Authenticate;
            proxy_set_header Authorization $http_authorization;
#            proxy_no_cache $cookie_nocache  $arg_nocache$arg_comment;
#            proxy_no_cache $http_pragma     $http_authorization;
#            proxy_cache_bypass $cookie_nocache $arg_nocache $arg_comment;
#            proxy_cache_bypass $http_pragma $http_authorization;
            client_max_body_size 50m;
            client_body_buffer_size 256k;
            proxy_connect_timeout 60;
            proxy_send_timeout 60;
            proxy_read_timeout 120;
            proxy_buffer_size 24k;
            proxy_buffers 4 64k;
            proxy_busy_buffers_size 64k;
            proxy_temp_file_write_size 64k;
            proxy_max_temp_file_size 128m;
            
            proxy_pass http://dwserver;
        }

        access_log /data1/app/log/nginx/dwout.test.com.log combined;
        error_log  /data1/app/log/nginx/error-dwout.test.com.log warn;
   }
```