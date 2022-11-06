---
title: 使用MySecureShell做sftp服务器安装与配置
author: 阿辉
date: 2015-05-07T09:52:32+00:00
categories:
- Linux
tags:
- Linux
- MySecureShell
keywords:
- Linux
- MySecureShell
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
有些合作方需要我们提供sftp服务帐号来交换信息，linux自带的sftp开出去不是很安全，网上找了一下，发现了MySecureShell这个东东作sftp服务器非常不错。

# 1. 安装
下载页：
http://mysecureshell.sourceforge.net/en/download.html
```
wget http://mysecureshell.free.fr/repository/index.php/centos/6.4/mysecureshell-1.33-1.x86_64.rpm
rpm -ivh mysecureshell-1.33-1.x86_64.rpm
```

# 2. 配置

<!--more-->

```bash
vim /etc/ssh/sftp_config 
## MySecureShell Configuration File ##
#Default rules for everybody
<Default>
        GlobalDownload          0       #total speed download for all clients
                                        # o -> bytes   k -> kilo bytes   m -> mega bytes
        GlobalUpload            0       #total speed download for all clients (0 for unlimited)
        Download                0       #limit speed download for each connection
        Upload                  0       #unlimit speed upload for each connection
        StayAtHome              true    #limit client to his home
        VirtualChroot           true    #fake a chroot to the home account
        LimitConnection         100     #max connection for the server sftp
        LimitConnectionByUser   50      #max connection for the account
        LimitConnectionByIP     5       #max connection by ip for the account
        Home                    /home/$USER     #overrite home of the user but if you want you can use
                                                #       environment variable (ie: Home /home/$USER)
        IdleTimeOut             5m      #(in second) deconnect client is idle too long time
        ResolveIP               true    #resolve ip to dns
#       IgnoreHidden            true    #treat all hidden files as if they don't exist
#       DirFakeUser             true    #Hide real file/directory owner (just change displayed permissions)
#       DirFakeGroup            true    #Hide real file/directory group (just change displayed permissions)
#       DirFakeMode             0400    #Hide real file/directory rights (just change displayed permissions)
                                        #Add execution right for directory if read right is set
        HideNoAccess            true    #Hide file/directory which user has no access
#       MaxOpenFilesForUser     20      #limit user to open x files on same time
#       MaxWriteFilesForUser    10      #limit user to x upload on same time
#       MaxReadFilesForUser     10      #limit user to x download on same time
        DefaultRights           0644 0755       #Set default rights for new file and new directory
#       MinimumRights           0400 0700       #Set minimum rights for files and dirs

        ShowLinksAsLinks        false   #show links as their destinations
#       ConnectionMaxLife       1d      #limits connection lifetime to 1 day

#       Charset                 "ISO-8859-15"   #set charset of computer
</Default>

#Rules only for group ftp
#<Group ftp>
#       Download        25 k/s
#       LogFile         /var/log/sftp-server_ftp.log    #Change logfile
#       ExpireDate      "2007-02-28 18:31:01"
#</Group>

#<Group sftp_administrator>
#       IsAdmin         true            #can admin the server
#       VirtualChroot   false           #you must disable chroot to have a full support of admin
#       StayAtHome      true
#       IdleTimeOut     0
#</Group>

#<Group old_client>
#       SftpProtocol            3       #force protocol SFTP
#       DisableAccount          true    #disable account
#</Group>

#Rules only for group ftpnolimit
#<Group ftpnolimit>
#       Download                0       #0 = unlimited
#       IdleTimeOut             0       #no timeout
#       DirFakeUser             false   #show real user on file/directory
#       DirFakeGroup            false   #show real group on file/directory
#       DirFakeMode             0       #show real rights on file/directory
#       MaxReadFilesForUser     0       #0 = unlimited but still have the restriction MaxOpenFilesForUser
#</Group>

#<IpRange 192.168.0.1-192.168.0.5>
#       ByPassGlobalDownload    true    #bypass GlobalDownload restriction
#       ByPassGlobalUpload      true    #bypass GlobalUpload restriction
#       Download                0
#       DisableAccount          false   #enable account
#       IdleTimeOut             0       #disable timeout
#       LimitConnectionByIP     0       #no limit
#</IpRange>

#<Group trusted_users>
#       Shell           /bin/tcsh       #give a shell access to TRUSTED clients !!!
#</Group>

#<VirtualHost *:22> 
#       DirFakeUser     false   #show real user on file/directory
#       DirFakeGroup    false   #show real group on file/directory
#       DirFakeMode     0       #show real rights on file/directory
#       HideNoAccess    false
#       IgnoreHidden    false
#</VirtualHost>

#Include /etc/my_sftp_config_file       #include this valid configuration file
```
配置都有详细的介绍，都不解释了，可以参考官方文档：
https://mysecureshell.readthedocs.org/en/latest/

# 3. 管理

## 1) 使用sftp-state启动和停止：
```
[root@ftp-lan sftpserver]# sftp-state  help
Usage:
------

sftp-state {options} {states}


Options:
        -yes : assume yes to all questions

States:
        - active : wake up server
        - start : same as 'active'
        - shutdown : shutdown the server (but don't kill current connections)
        - stop : same as 'shutdown'
        - fullstop : shutdown the server (kill all connections and clean memory)
[root@ftp-lan sftpserver]# sftp-state
Server is up
[root@ftp-lan sftpserver]# sftp-state stop
Shutdown server for new connection (active connection are keeped)
Do you want to kill all users ? [YES/no] yes
[root@ftp-lan sftpserver]# sftp-state start
Server is now online.
[root@ftp-lan sftpserver]#
```

## 2) 创建与删除用户
```
[root@ftp-lan sftpserver]# sftp-user create test
Enter password:

[root@ftp-lan sftpserver]# sftp-user delete test
```

# 4. 测试

从其它机器上测试
```
[root@fft-vm-newapp-2 ~]# sftp test@172.22.2.2
Connecting to 172.22.2.2...
test@172.22.2.2's password: 
sftp> ls
sftp> ls /
sftp> mkdir aaa
sftp> exit
[root@fft-vm-newapp-2 ~]# ssh test@172.22.2.2 
test@172.22.2.2's password: 
Shell access is disabled !Connection to 172.22.2.2 closed.
```
可以看到sftp连上后有chroot，并而对这个用户关闭了shell.

另外openssh本身也是可以实现chroot的，只是功能稍为差一些，MySecureShell有限制流量，并发等好多功能。可以参考：
http://www.mike.org.cn/articles/centos-sftp-chroot/
