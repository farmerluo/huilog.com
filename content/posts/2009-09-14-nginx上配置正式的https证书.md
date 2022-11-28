---
title: nginx上配置正式的https证书
author: 阿辉
date: 2009-09-14T11:07:00+00:00
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
nginx上配置https证书与apache有些不一样，参考：http://wiki.nginx.org/NginxHttpSslModule，如果是自己建一个证书用，方法如下：
```
Generate Certificates

To generate dummy certficates you can do this steps:

cd /usr/local/nginx/conf
openssl genrsa -des3 -out server.key 1024
openssl req -new -key server.key -out server.csr
cp server.key server.key.org
openssl rsa -in server.key.org -out server.key
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
Configure the new certificate into nginx.conf:

server {
    server_name YOUR_DOMAINNAME_HERE;
    listen 443;
    ssl on;
    ssl_certificate /usr/local/nginx/conf/server.crt;
    ssl_certificate_key /usr/local/nginx/conf/server.key;
}
Restart Nginx.

Now all ready to access using:

https://YOUR_DOMAINNAME_HERE
```
上面这种自己建的证书用firefox打开会有个提示：

YOUR_DOMAINNAME_HERE 使用了无效的安全证书。该证书仅对下列名称有效:

（错误码： ssl_error_bad_cert_domain）

<!--more-->

这是因为证书并不是经过认证的，希望IE或firefox不提示错误，需要向发证机构申请证书。我们在WoSign上申请了一个证书。
申请步骤如下：

1）先生成一个key:
```bash
[root@ha1 sslkey]# openssl genrsa -des3 -out server.key 1024
Generating RSA private key, 1024 bit long modulus
……………………………………………………………..++++++
.++++++
e is 65537 (0x10001)
Enter pass phrase for server.key:
Verifying - Enter pass phrase for server.key:
[root@ha1 sslkey]# ls
server.key
```
会提示输入一个密码，这个要记好。

2）生成CSR:
```bash
[root@ha1 sslkey]# openssl req -new -key server.key -out server.csr
Enter pass phrase for server.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter ‘.’, the field will be left blank.
—–
Country Name (2 letter code) [GB]:CN
State or Province Name (full name) [Berkshire]:ShangHai
Locality Name (eg, city) [Newbury]:ShangHai
Organization Name (eg, company) [My Company Ltd]:Shine Zone Co., Ltd.
Organizational Unit Name (eg, section) []:IT
Common Name (eg, your name or your server’s hostname) []:www.domain.com
Email Address []:admin@shinezone.com

Please enter the following ‘extra’ attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
输入刚生成key的密码及相关信息。就可以生成一个csr文件了。
然后把生成的server.csr给WoSign就可以申请证书了。

申请成功后，他们给我两个文件，server.ca-bundle及server.crt，按手册配置如下：
```
server {
    server_name YOUR_DOMAINNAME_HERE;
    listen 443;
    ssl on;
    ssl_certificate /usr/local/nginx/conf/server.crt;
    ssl_certificate_key /usr/local/nginx/conf/server.key;
}
```
用IE访问没有提示了，但用firefox访问时，提示：https sec_error_unknown_issuer，这个问题的原因是由于nginx没有配置中级根证书造成的。解决方法如下：
```bash
cat server.crt server.ca-bundle > server.pem

修改配置文件：
server {
  server_name YOUR_DOMAINNAME_HERE;
  listen 443;
  ssl on;
  ssl_certificate /usr/local/nginx/conf/server.pem;
  ssl_certificate_key /usr/local/nginx/conf/server.key;
}
```
再重启就测试就不会有错误提示了。

因为之前我们在生成key的时候用了密码。你会发现配置好nginx证书后，在重启nginx时，会提示你输入密码，输入密码后才能启动。如何取消证书的密码呢？

有两种方式：
1）把带密码的key转成不带密码的：
`openssl rsa -in server.key -out server2.key`

2) 在第一步生成key时不带-des3参数：
`openssl genrsa -out www_shinezone_com.key 1024`
