---
title: nginx+php-fpm中使用alias需要注意的地方
author: 阿辉
date: 2014-10-10T06:31:58+00:00
categories:
- Nginx
tags:
- Nginx
- PHP
keywords:
- Nginx
- PHP
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
今天在配置zabbix，之前zabbix是使用apache+php的，现在想换成nginx+php-fpm，nginx配置如下：
```
        location /zabbix/ {

            alias           /usr/share/zabbix/;
            index           index.php;
            error_page      403 404 502 503 504  /zabbix/index.php;

            location ~ .php$ {
                expires        epoch;
                fastcgi_pass   unix:/tmp/php-cgi.sock;
                fastcgi_index  index.php;
                include        fastcgi.conf;
            }

            location ~ .(jpg|jpeg|gif|png|ico)$ {
                access_log  off;
                expires     33d;
            }

        }
```
发现通过WEB访问zabbix PHP程序时，显示是404未找到文件的错误。

<!--more-->

fastcgi.conf文件如下：
```
fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;

fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  HTTPS              $https if_not_empty;

fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;

fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;

# PHP only, required if PHP was built with --enable-force-cgi-redirect
fastcgi_param  REDIRECT_STATUS    200;
```
发现其中SCRIPT_FILENAME有点问题：

```
fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
```
可以看到：SCRIPT_FILENAME是由$document_root和$fastcgi_script_name组成的，而$document_root应该是nginx配置文件内的`root /xx/xxx/`，而我们使用了alias改到其它路径了，所以会找不到php文件。

修改nginx配置文件，加上一行:
`fastcgi_param  SCRIPT_FILENAME    $fastcgi_script_name;`
替换掉fastcgi.conf内的SCRIPT_FILENAME变量配置，再测试就OK了。

能正常运行的nginx配置如下：
```
        location /zabbix/ {

            alias           /usr/share/zabbix/;
            index           index.php;
            error_page      403 404 502 503 504  /zabbix/index.php;

            location ~ .php$ {
                expires        epoch;
                fastcgi_pass   unix:/tmp/php-cgi.sock;
                fastcgi_index  index.php;
                include        fastcgi.conf;
                fastcgi_param  SCRIPT_FILENAME $request_filename;
            }

            location ~ .(jpg|jpeg|gif|png|ico)$ {
                access_log  off;
                expires     33d;
            }

        }
```