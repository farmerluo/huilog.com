---
title: 升级qmail时导入旧qmail系统用户及邮件的脚本
author: 阿辉
date: 2009-07-22T15:32:00+00:00
categories:
- Qmail
tags:
- Qmail
keywords:
- Qmail
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

编写了一个升级qmail时导入旧qmail系统用户及邮件的脚本：

<!--more-->
```bash
#!/bin/bash
#本脚本用于升级qmail时导入旧的qmail系统的用户及邮件
#auther:阿辉
#date:2009.06.12

adduser=”/home/vpopmail/bin/vadduser”
vuserinfo=”/home/vpopmail/bin/vuserinfo”
quota=”100MB”
#vpopmail.csv为旧的vpopmail数据库表的导出文件
vpopmail=”/root/vpopmail.csv”

sed ‘/^s/d’ vpopmail | sed ‘/^s*#./d’ | while read list
do
    echo
    echo
    username=echo list | cut -d ';' -f 2 | tr -d '"'
    domain=echo list | cut -d ';' -f 3 | tr -d '"'
    encrypt=echo list | cut -d ';' -f 4 
    password=echo list | cut -d ';' -f 10 | tr -d '"'
    comment=echo list | cut -d ';' -f 7 
    dir=”/opt”echo list | cut -d ';' -f 8 | tr -d '"'
    email=username”@”domain

    #建用户
    echo adduser -q quota -c comment email password
    adduser -q quota email password
   
    #更新用户信息及加密密码
    echo “update vpopmail.fortelchina_com set pw_passwd = encrypt,pw_gecos = comment where pw_name = ‘username’”
    echo “update vpopmail.fortelchina_com set pw_passwd = encrypt,pw_gecos = comment where pw_name = ‘username’” | mysql -uroot -p123456

    #导入用户邮件
    pw_dir=echo "select pw_dir from vpopmail.fortelchina_com where pw_name = 'username'" | mysql -uroot -p123456 | sed '1d'
    echo /bin/cp -rf dir/Maildir pw_dir
    /bin/cp -rf dir/Maildir pw_dir
   
    #修改邮件的权限
    echo chown -R vpopmail.vchkpw pw_dir
    chown -R vpopmail.vchkpw pw_dir

    #重新生成maildirsize文件
    echo rm -rf pw_dir/Maildir/maildirsize
    rm -rf pw_dir/Maildir/maildirsize
    echo vuserinfo email
    vuserinfo email
   
done
```