---
title: 利用dnsmasq来修改dns记录
author: 阿辉
date: 2014-08-11T14:27:36+00:00
categories:
- Linux
tags:
- Dnsmasq
- Linux
keywords:
- Dnsmasq
- Linux
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
我想做技术的或多或少都会知道dnsmasq，因为基本上所有的家用路由都内置了它，使用他来做域名服务的缓存代理。

仔细研究你会发现，除了缓存代理的基本功能之外，它修改dns记录的功能也很有意思。看看下面：

# 1. 安装dnsmasq:

在centos linux上的，可以直接yum安装：
```
yum install -y dnsmasq
```

# 2. 配置dnsmasq:  
vim /etc/dnsmasq.conf  
```
#监听所有网卡  
bind-interfaces  
#修改A记录  
address=/www.163.com/1.2.3.4  
#修改mx记录  
mx-host=126.com,m.126.com,10  
mx-host=163.com,m.163.com,10
```
<!--more-->

保存配置文件，重启dnsmasq:
```
[root@oracle ~]# service dnsmasq restart  
Shutting down dnsmasq:                                     [  OK  ]  
Starting dnsmasq:                                                 [  OK  ]
```

# 3. 用dig查询一下结果
```
[root@oracle ~]# dig @127.0.0.1 www.163.com

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> @127.0.0.1 www.163.com  
; (1 server found)  
;; global options: +cmd  
;; Got answer:  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 56629  
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:  
;www.163.com.                   IN      A

;; ANSWER SECTION:  
www.163.com.            0       IN      A       1.2.3.4

;; Query time: 0 msec  
;; SERVER: 127.0.0.1#53(127.0.0.1)  
;; WHEN: Mon Aug 11 22:25:17 2014  
;; MSG SIZE  rcvd: 45

[root@oracle ~]# dig @127.0.0.1 163.com mx

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> @127.0.0.1 163.com mx  
; (1 server found)  
;; global options: +cmd  
;; Got answer:  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4324  
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:  
;163.com.                       IN      MX

;; ANSWER SECTION:  
163.com.                0       IN      MX      10 m.163.com.

;; Query time: 0 msec  
;; SERVER: 127.0.0.1#53(127.0.0.1)  
;; WHEN: Mon Aug 11 22:25:21 2014  
;; MSG SIZE  rcvd: 50

[root@oracle ~]# dig @127.0.0.1 126.com mx

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> @127.0.0.1 126.com mx  
; (1 server found)  
;; global options: +cmd  
;; Got answer:  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 50441  
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:  
;126.com.                       IN      MX

;; ANSWER SECTION:  
126.com.                0       IN      MX      10 m.126.com.

;; Query time: 0 msec  
;; SERVER: 127.0.0.1#53(127.0.0.1)  
;; WHEN: Mon Aug 11 22:25:25 2014  
;; MSG SIZE  rcvd: 50

[root@oracle ~]# dig @127.0.0.1 sina.com.cn mx

; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.17.rc1.el6_4.6 <<>> @127.0.0.1 sina.com.cn mx  
; (1 server found)  
;; global options: +cmd  
;; Got answer:  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 34068  
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:  
;sina.com.cn.                   IN      MX

;; ANSWER SECTION:  
sina.com.cn.            181     IN      MX      10 freemx2.sinamail.sina.com.cn.  
sina.com.cn.            181     IN      MX      5 freemx.sinamail.sina.com.cn.  
sina.com.cn.            181     IN      MX      10 freemx1.sinamail.sina.com.cn.  
sina.com.cn.            181     IN      MX      10 freemx3.sinamail.sina.com.cn.

;; Query time: 6 msec  
;; SERVER: 127.0.0.1#53(127.0.0.1)  
;; WHEN: Mon Aug 11 22:25:40 2014  
;; MSG SIZE  rcvd: 133

```

至于这个功能有什么用处?不用我说了，你懂的。。。