---
title: 用denyhosts来防止ssh攻击
author: 阿辉
date: 2006-09-29T11:19:00+00:00
categories:
- Linux
tags:
- Linux
keywords:
- Linux
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
每天服务器都遭受着上千次的SSH失败的尝试：
```bash
sshd:
   Authentication Failures:
      root (218.247.185.218): 575 Time(s)
      unknown (218.247.185.218): 224 Time(s)
      unknown (218.247.185.222): 6 Time(s)
      unknown (202.101.72.35): 5 Time(s)
      unknown (202.101.72.36): 5 Time(s)
      unknown (202.101.72.37): 5 Time(s)
      unknown (202.101.72.44): 5 Time(s)
      unknown (202.101.72.32): 4 Time(s)
      unknown (202.101.72.40): 4 Time(s)
      unknown (202.101.72.43): 4 Time(s)
      unknown (202.101.72.45): 4 Time(s)
      unknown (202.101.72.47): 4 Time(s)
      unknown (202.101.72.50): 4 Time(s)
      unknown (202.101.72.53): 4 Time(s)
      unknown (202.101.72.56): 4 Time(s)
      unknown (202.101.72.57): 4 Time(s)
      unknown (202.101.72.60): 4 Time(s)
      unknown (202.101.72.62): 4 Time(s)
      root (218.247.185.222): 3 Time(s)
      unknown (202.101.72.33): 3 Time(s)
      unknown (202.101.72.34): 3 Time(s)
      unknown (202.101.72.38): 3 Time(s)
      unknown (202.101.72.39): 3 Time(s)
      unknown (202.101.72.41): 3 Time(s)
      unknown (202.101.72.48): 3 Time(s)
      unknown (202.101.72.51): 3 Time(s)
      unknown (202.101.72.52): 3 Time(s)
      unknown (202.101.72.54): 3 Time(s)
      unknown (202.101.72.55): 3 Time(s)
      unknown (202.101.72.58): 3 Time(s)
      unknown (202.101.72.61): 3 Time(s)
      unknown (202.101.72.63): 3 Time(s)
      ftp (202.101.72.34): 2 Time(s)
      mail (218.247.185.218): 2 Time(s)
      mysql (218.247.185.218): 2 Time(s)
      news (218.247.185.218): 2 Time(s)
      root (192.168.123.69): 2 Time(s)
      unknown (202.101.72.42): 2 Time(s)
      unknown (202.101.72.46): 2 Time(s)
      unknown (202.101.72.49): 2 Time(s)
      unknown (202.101.72.59): 2 Time(s)
      adm (202.101.72.34): 1 Time(s)
      adm (202.101.72.42): 1 Time(s)
      adm (202.101.72.46): 1 Time(s)
      adm (202.101.72.49): 1 Time(s)
      adm (202.101.72.51): 1 Time(s)
      adm (202.101.72.58): 1 Time(s)
      adm (202.101.72.59): 1 Time(s)
      adm (202.101.72.61): 1 Time(s)
      adm (218.247.185.218): 1 Time(s)
      apache (218.247.185.218): 1 Time(s)
      bin (218.247.185.218): 1 Time(s)
      ftp (202.101.72.33): 1 Time(s)
      ftp (202.101.72.39): 1 Time(s)
      ftp (202.101.72.46): 1 Time(s)
      ftp (202.101.72.58): 1 Time(s)
      ftp (202.101.72.60): 1 Time(s)
      ftp (218.247.185.218): 1 Time(s)
      games (218.247.185.218): 1 Time(s)
      lp (218.247.185.218): 1 Time(s)
      mysql (202.101.72.38): 1 Time(s)
      mysql (202.101.72.39): 1 Time(s)
      mysql (202.101.72.42): 1 Time(s)
      mysql (202.101.72.49): 1 Time(s)
      mysql (202.101.72.51): 1 Time(s)
      mysql (202.101.72.59): 1 Time(s)
      mysql (202.101.72.61): 1 Time(s)
      nobody (218.247.185.218): 1 Time(s)
      operator (218.247.185.218): 1 Time(s)
      postgres (202.101.72.33): 1 Time(s)
      postgres (202.101.72.48): 1 Time(s)
      postgres (202.101.72.49): 1 Time(s)
      postgres (202.101.72.52): 1 Time(s)
      postgres (202.101.72.53): 1 Time(s)
      postgres (202.101.72.54): 1 Time(s)
      rpm (218.247.185.218): 1 Time(s)
      squid (218.247.185.218): 1 Time(s)
      sshd (218.247.185.218): 1 Time(s)
   Invalid Users:
      Unknown Account: 341 Time(s)
```

<!--more-->
今天没事在 http://dag.wieers.com/home-made/apt/packages.php 看到一个软件denyhosts，正好可以解决这个问题。

先安装一个包，以便用yum直接在dag上取包：
```bash
wget http://ftp.belnet.be/packages/dries.ulyssis.org/redhat/el4/en/i386/RPMS.dries/rpmforge-release-0.2-2.2.el4.rf.i386.rpm
rpm -ivh rpmforge-release-0.2-2.2.el4.rf.i386.rpm
```

这样就可以直接用yum安装denyhosts了：

`yum install denyhosts`

再进行一下设置：
```bash
cp /usr/share/doc/denyhosts-2.2/daemon-control-dist /etc/init.d/denyhosts
cp /usr/share/doc/denyhosts-2.2/denyhosts.cfg-dist /etc/denyhosts.cfg
vi /etc/init.d/denyhosts
# 将DENYHOSTS_CFG参数的值改成 “/etc/denyhosts.cfg”
```

再增加到services:
```bash
chkconfig –add denyhosts
chkconfig –level 2345 denyhosts on
```

再修改一下配置文件：
```bash
vi /etc/denyhosts.cfg

SECURE_LOG = /var/log/secure
#ssh 日志文件，它是根据这个文件来判断的。

HOSTS_DENY = /etc/hosts.deny
#控制用户登陆的文件

PURGE_DENY = 5m
#过多久后清除已经禁止的

BLOCK_SERVICE  = sshd
#禁止的服务名

DENY_THRESHOLD_INVALID = 1
#允许无效用户失败的次数

DENY_THRESHOLD_VALID = 10
#允许普通用户登陆失败的次数

DENY_THRESHOLD_ROOT = 5
#允许root登陆失败的次数

HOSTNAME_LOOKUP=NO
#是否做域名反解

ADMIN_EMAIL = hui@ffccc.com
#管理员邮件地址,它会给管理员发邮件

DAEMON_LOG = /var/log/denyhosts
#自己的日志文件
```

然后就可以启动了：

`service denyhost start`

可以看看`/etc/hosts.deny`内是否有禁止的ＩＰ，有的话说明已经成功了。