---
title: nagios改用nginx+fast-cgi模式运行
author: 阿辉
date: 2012-02-16T17:45:00+00:00
categories:
- Nagios
tags:
- Nagios
keywords:
- Nagios
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
现在apache用得越来越少了，大家都改用nginx。但有些东西还是比较依赖apache,如nagios。

想让nagios在nginx上运行，必需先让nginx支持perl和cgi解析的功能，需要用到fcgi-perl,安装方式参见我的上一篇blog：

[在nginx上配置使用bugzilla](/2012/01/在nginx上配置使用bugzilla/)

在这里就不多说了。

<!--more-->

现在帖上perl的fast-cgi起来后，nginx的配置：
```
   server {
        listen       80;
        server_name  monitor.xxxx.com;

        root   /data1/www/monitor.xxxx.com;
        index  index.php index.html index.htm;

        access_log /data1/app/log/nginx/monitor.xxxx.com.log  combined;
        error_log  /data1/app/log/nginx/error-monitor.xxxx.com.log notice;

        allow 10.0.0.0/8;
        deny all;

        location ~ .php$ {

            root  /data1/www/monitor.xxxx.com;

            fastcgi_pass   unix:/data1/app/tmp/php-cgi.sock;
            fastcgi_index  index.php;
            include        fastcgi.conf;
        }

        location /nagios/ {

            alias /usr/share/nagios/;
            index index.html index.htm index.php;

            auth_basic "Nagios Access";
            auth_basic_user_file htpasswd.users;

            location ~ .php$ {
                root /usr/share;
                fastcgi_pass   unix:/data1/app/tmp/php-cgi.sock;
                fastcgi_index  index.php;
                include        fastcgi.conf;
            }

        }

        location ~ .*.(pl|cgi)$ {
            rewrite ^/nagios/cgi-bin/(.*).cgi /$1.cgi break;

            auth_basic "Nagios Access";
            auth_basic_user_file htpasswd.users;

            gzip off;
            include fastcgi_params;
            fastcgi_pass  127.0.0.1:8999;
            fastcgi_index index.cgi;
            fastcgi_param SCRIPT_FILENAME  /usr/lib64/nagios/cgi$fastcgi_script_name;
            fastcgi_param AUTH_USER $remote_user;
            fastcgi_param REMOTE_USER $remote_user;

        }

   }
```

然后重新生成认证文件htpasswd.users放在nginx的conf目录。重启nginx服务便可。

生成认证文件使用：
```
htpasswd -c htpasswd.users nagiosadmin
```
特别注意下面两个参数，一定要加上：
```
            fastcgi_param AUTH_USER $remote_user;
            fastcgi_param REMOTE_USER $remote_user;
```
否则进入nagios会提示没有认证。