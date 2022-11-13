---
title: 怎么在amazon AWS配置主机名
author: 阿辉
date: 2012-03-21T16:16:00+00:00
categories:
- AWS
tags:
- AWS
- EC2
- Vsftpd
keywords:
- AWS
- EC2
- Vsftpd
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
想想这个问题真可笑，配置一个主机名还能写一篇博客。

这些都是因为AWS的IP分配方式造成的，AWS不管公网IP还是私有IP都是自动分配的方式。

所以开机后域名就会自动设成ip的形式,如：
```
ip-10-168-23-240
```
AWS实例关机再重开后，上面这个地址就会变成另外一个名字。对我们的服务器管理造成一些麻烦。

<!--more-->

基于此，AWS也提出了一些解决办法，给每个实例一份metadata和一份userdata，通过下面命令可以看到这些数据：

Amazon Docs :

http://docs.amazonwebservices.com/AWSEC2/2007-03-01/DeveloperGuide/AESDG-chapter-instancedata.html

先看meta-data：
```
[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/
ami-id
ami-launch-index
ami-manifest-path
block-device-mapping/
hostname
instance-action
instance-id
instance-type
local-hostname
local-ipv4
mac
metrics/
network/
placement/
profile
public-hostname
public-ipv4
public-keys/
reservation-id
security-groups
```

上面这些东西也都是可以通过http的方式查看的，如：
```
[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/local-hostname
ip-10-168-23-240.us-west-1.compute.internal

[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/hostname      
ip-10-168-23-240.us-west-1.compute.internal

[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/network/
interfaces/

[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/network/interfaces/
macs/

[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/network/interfaces/macs/
12:31:3f:04:14:02/

[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/meta-data/public-ipv4
50.38.68.1
```
这样就可以保证我们在实例开机后，可以查看公有及私有IP等信息。

再看看用户数据(user data)，用户数据是需要开机前或实例创建时由用户自己去填写的数据，填写后可以通过上面类似的方式查出来。

如我在实例启用前配置了user data为test，那么执行：
```
[root@ip-10-168-23-240 ~]# curl http://169.254.169.254/latest/user-data
test
```
好，现在有主机名和公有及私有IP，就可以通过这些信息配置主机名。机器一多，手动改这些就不太现实，用脚本来实现：
```
[root@ip-10-168-23-240 ~]# vim /usr/local/ec2/ec2-hostname.sh

#!/bin/bash

# Replace this with your domain
DOMAIN=your-domain.com

USER_DATA=/usr/bin/curl -s http://169.254.169.254/latest/user-data
HOSTNAME=echo $USER_DATA
IPV4=/usr/bin/curl -s http://169.254.169.254/latest/meta-data/public-ipv4

# Set the host name
hostname $HOSTNAME
echo $HOSTNAME > /etc/hostname

# Add fqdn to hosts file
cat<<EOF > /etc/hosts
# This file is automatically genreated by ec2-hostname script
127.0.0.1 localhost
$IPV4 $HOSTNAME.$DOMAIN $HOSTNAME

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts

EOF
[root@ip-10-168-23-240 ~]# chmod o+x ec2-hostanmes.sh
```
把这个脚本加入到/etc/rc.local，开机自动执行。

这样下次重启后就可以自动配置一个主机名了。但是千万不要忘记配置实例的userdata.

其实用脚本的方式还是有些问题，就是主机名虽然加上了，但是用在其它机器上ping这个主机名是得不到IP的。因为经常变，也不方便手动给域名加IP。

下面这些文章介绍了用bind的域名update的方式自动修改主机名对应的IP，是一个不错的解决办法。烦的是就是要自己维护一个域名服务器了。

http://www.ducea.com/2009/06/01/howto-update-dns-hostnames-automatically-for-your-amazon-ec2-instances/