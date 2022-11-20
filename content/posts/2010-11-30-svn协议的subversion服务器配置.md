---
title: svn协议的subversion服务器配置
author: 阿辉
date: 2010-11-30T13:57:00+00:00
categories:
- SVN
tags:
- SVN
keywords:
- SVN
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
经过几天svn+ssh的使用，最终大家还是受不了svn+ssh的麻烦和速度，在另一个机房又架了台svn协议的subversion服务器。

哈哈，记录下配置过程：

1) 安装ssh server和subversion
```bash
yum install -y openssh-server subversion
```

2) 建立 subversion repository
```bash
mkdir /var/svn-repos
svnadmin create /var/svn-repos/test
```

3) 启动服务：
```bash
vi /etc/xinetd.d/svn
service svn
{
        disable                 = no
        port                    = 3690
        socket_type             = stream
        protocol                = tcp
        wait                    = no
        user                    = root
        server                  = /usr/bin/svnserve
        server_args             = -i -r /var/svn-repos
}

service xinetd restart
```

4) 修改repository配置，并启用authz权限控制
```bash
vi /var/svn-repos/test/conf/svnserve.conf
#在general小节中，加入几行内容
anon-access = none
auth-access = write
password-db = passwd
authz-db = authz
```

5) 加用户：
```bash
vi /var/svn-repos/test/conf/passwd
user = password
```

6) 设权限：
```bash
vi /var/svn-repos/test/conf/authz
[/]
user = rw
```

# 导入方法：
svn import web svn://192.168.1.10/test -m “initial import”

# 检出方法：
svn co svn://192.168.1.10/test
```

其它同步脚本等配置同前文介绍。