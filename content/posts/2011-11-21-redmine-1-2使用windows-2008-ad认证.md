---
title: redmine 1.2使用windows 2008 AD认证
author: 阿辉
date: 2011-11-21T13:01:00+00:00
categories:
- Redmine
tags:
- Redmine
keywords:
- Redmine
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
这几天在弄redmine通过windows 2008域服务器的AD认证问题，目的是让redmine去读AD上的用户帐号，自动注册和登陆。

redmine的配置如下：

![/wp-content/uploads/baiduhi/709bc93d70cf3bc749b62cabd100baa1cc112a51.jpg](/wp-content/uploads/baiduhi/709bc93d70cf3bc749b62cabd100baa1cc112a51.jpg)

以上配置区分大小写，需要注册的有帐号和Base DN这两个地方，填错了就会登陆不了。登陆时提示： 无效的用户名或密码

比如域服务器的域名为：host.domain.com，

帐号需配置成这样：CN=administrator,CN=Users,DC=host,DC=domain,DC=com

Base DN：DC=host,DC=domain,DC=com

我的经验是DN不需要加CN=Users，加了之后只有下图中Users下的用户可以登陆，而Domain Controllers下的用户将登陆不了。

<!--more-->

![/wp-content/uploads/baiduhi/51b109f7905298226922018bd7ca7bcb0b46d4f2.jpg](/wp-content/uploads/baiduhi/51b109f7905298226922018bd7ca7bcb0b46d4f2.jpg)

帐号一定要配置成上面这样，类似host.domain.comadministrator或administrator，经过我的测试是不行的。帐号也不一定非要administrator,只要有查看域权限的用户就行。

主机填AD服务器的IP,端口默认为389。

密码是指administrator用户的密码。

配置好之后，保存。再用ad的用户直接登陆redmine,可以登陆就说明成功了。

另外还需要注意的是：在redmine早一点的版本中，如果AD内的用户没有配置mail或姓名之类的资料，也是登陆不了的。不过在我测试的1.2版本上，如果mail不对或重复，用户在登陆redmaine时，redmine将会让用户重新填写。

linux的openldap有一个测试ldap的命令，叫：ldapsearch，配置不成功可以用这个命令测试，如：

ldapsearch -x -h host.domain.com -b "dc=host,dc=domain,dc=com" -D"cn=administrator,cn=users,dc=host,dc=domain,dc=com" -w password > ad.txt

ldapsearch参考：
http://godoha.blog.51cto.com/108180/73382
