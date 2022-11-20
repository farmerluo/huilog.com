---
title: GlusterFS分布式集群文件系统安装、配置及性能测试
author: 阿辉
date: 2010-05-17T10:38:00+00:00
categories:
- GlusterFS
tags:
- GlusterFS
keywords:
- GlusterFS
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
{{< toc >}}

前几天和天涯的刘天斯在讨论分布式文件系统，才想起电脑内还有一篇一年前写的文档，现在帖在这里，给有需要的朋友看看，因为当时是用word写的，帖在这边排版不是很好。大家凑合着看吧。

# 1. 版本历史
```
Revision    Author(s)    Date    Summary of activity
1.0         罗辉         2009-6-1    创建本文档
```
# 2. 参考文档

[1] http:// www.gluster.org

[2] http://wenzizone.cn/?p=8

# 3. 前言

Glusterfs是一个具有可以扩展到几个PB数量级的分布式集群文件系统。它可以把多个不同类型的存储块通过Infiniband RDMA或者TCP/IP汇聚成一个大的并行网络文件系统。

考虑到公司图片服务器后期的升级，我们对Glusterfs进行了比较详细的技术测试。

<!--more-->

# 4. 测试环境

我们采用八台老的至强服务器组成了测试环境，配置为内存1-2G不等，每台有两块以上的73G SCSI硬盘。

同时每服务器配有两块网卡，连接至两个100M以太网交换机上。192.168.1.x段连接的是cisco 2950，另一个段是一个d-link交换机，服务器间的传输主要是通过cisco 2950，以保证网络的稳定性。

IP地址分别为:192.168.1.11,192.168.1.18 及 192.168.190.11,192.168.19018。

所有服务器的操作系统都是Centos linux 5.3，安装DAG RPM Repository的更新包。DAG RPM Repository下载页面为：http://dag.wieers.com/rpm/packages/rpmforge-release/。
安装方式：
```bash
# wget http://dag.wieers.com/rpm/packages/rpmforge-release/rpmforge-release-0.3.6-1.el5.rf.i386.rpm
# rpm –ivh rpmforge-release-0.3.6-1.el5.rf.i386.rpm
```

# 5. GlusterFS的安装

## 5.1. 服务器端安装

我们通过rpm编译方式来安装GlusterFS，因为做为群集文件系统，可能需要在至少10台以上的服务器上安装GlusterFS。每台去源码编译安装太费功夫，缺乏效率。在一台编译为rpm包，再复制到其它的服务器上安装是最好的选择。

GlusterFS需要fuse支持库，需先安装:
```bash
# yum -y install fuse fuse-devel httpd-devel libibverbs-devel
```
下载GlusterFS源码编译rpm包。
```bash
# wget http://ftp.gluster.com/pub/gluster/glusterfs/2.0/LATEST/glusterfs-2.0.0.tar.gz
# tar -xvzf glusterfs-2.0.0.tar.gz
# cp glusterfs-2.0.0.tar.gz /usr/src/redhat/SOURCES/
# rpmbuild -bb glusterfs-2.0.0/glusterfs.spec
# cp /usr/src/redhat/RPMS/i386/glusterfs* .
# rm glusterfs-debuginfo-2.0.0-1.i386.rpm
# rpm -ivh glusterfs-.rpm
```
安装完成，并把编译好的rpm包复制到其它服务器上安装。

# 5.2. 客户端安装

客户端和服务器有一点点不同，特别需要注意的是在客户端这边，不但需要fuse库，并且需要一个fuse内核模块。好在DAG RPM Repository内已有用DKMS方式编译好的内核模块包，我们直接安装便可。

DKMS(Dynamic Kernel Module Support)是dell发起的一个项目，目的是希望能在不编译内核的情况下，动态的更新内核模块，最重要的是，通过DKMS方式编译的内核模块，由于是由DKMS管理的，在内核升级后，无需重新编译，仍旧可用。这种方式可大大方便以后的内核更新。

GlusterFS可直接用上面rpm编译后的包安装:
```bash
# yum -y install dkms dkms-fuse fuse fuse-devel httpd-devel libibverbs-devel
# rpm -ivh glusterfs-*.rpm
```

# 6. GlusterFS的典型架构图

![/wp-content/uploads/baiduhi/90d6ebf8aa16de35d8f9fd43.jpg](/wp-content/uploads/baiduhi/90d6ebf8aa16de35d8f9fd43.jpg)

# 7. GlusterFS常用translators(中继)

## 7.1. storage/posix

type storage/posix

storage/posix的作用是指定一个本地目录给GlusterFS内的一个卷使用。

配置例子：
```bash
volume posix-example
type storage/posix
option directory /sda4
end-volume
```

## 7.2. protocol/server (服务器)

type protocol/server

服务器中继(protocol/server）表示本节点在GlusterFS中为服务器模式。

配置例子：
```bash
volume server-example
type protocol/server
option transport-type tcp
subvolumes brick                #定义好的卷
option auth.addr.brick.allow *  #指定可访问本卷的访问者,为所有，可对访问者做限制，如192.168.1.*
end-volume
```

## 7.3. protocol/client (客户端)

type protocol/client

客户端中继(protocol/server）用于客户端连接服务器时使用。

配置例子：
```bash
volume client-example
type protocol/client
option transport-type tcp
option remote-host 192.168.1.13    #连接的服务器
option remote-subvolume brick      #连接的服务器卷名
end-volume
```

## 7.4. cluster/replicate(复制)

type cluster/replicate

复制中继(cluster/replicate，前身是AFR）为GlusterFS提供了类似RAID-1的功能。

Replicate会复制文件或者文件夹到各个subvolumes里。如一个卷(volume)内有两个子卷(subvolume)，那就会有两份文件或文件夹的复本。

Replicate只时还有高可用的功能，如果两个子卷中有一个子卷挂了，卷依然可以正常工作。当这个子卷重新启用时，会自动更新丢失的文件或文件夹，不过更新是通过客户端进行的。

配置例子：
```bash
volume replicate-example
type cluster/replicate
subvolumes brick3 brick4
end-volume
```

## 7.5. cluster/distribute (分布式)

type cluster/distribute

分布式中继(cluster/distribute，前身是unify）为GlusterFS提供了类似RAID-0的功能。

Distribute可把两个卷或子卷组成一个大卷，实现多存储空间的聚合

配置例子：
```bash
volume distribute-example
type cluster/distribute
subvolumes repl1 repl2
end-volume
```

## 7.6. features/locks (锁)

type features/locks

锁中继(features/locks）只能用于服务器端的posix中继之上，表示给这个卷提供加锁(fcntl locking)的功能。

配置例子：
```bash
volume locks-example
type features/locks
subvolumes posix-example
end-volume
```

7.7. performance/read-ahead (预读)

type performance/read-ahead

预读中继(performance/read-ahead）属于性能调整中继的一种，用预读的方式提高读取的性能。

读取操作前就预先抓取数据。这个有利于应用频繁持续性的访问文件，当应用完成当前数据块读取的时候，下一个数据块就已经准备好了。

额外的，预读中继也可以扮演读聚合器，许多小的读操作被绑定起来，当成一个大的读请求发送给服务器。

预读处理有page-size和page-count来定义，page-size定义了，一次预读取的数据块大小，page-count定义的是被预读取的块的数量

不过官方网站上说这个中继在以太网上没有必要，一般都能跑满带宽。主要是在IB-verbs或10G的以太网上用。

配置例子：
```bash
volume readahead-example
type performance/read-ahead
option page-size  256         # 每次预读取的数据块大小
option page-count 4           # 每次预读取数据块的数量
option force-atime-update off #是否强制在每次读操作时更新文件的访问时间，不设置这个，访问时间将有些不精确，这个将影响预读转换器读取数据时的那一时刻而不是应用真实读到数据的那一时刻。
subvolumes
end-volume
```

## 7.8. performance/write-behind (回写)

type performance/write-behind

回写中继(performance/read-ahead）属于性能调整中继的一种，作用是在写数据时，先写入缓存内，再写入硬盘。以提高写入的性能。

回写中继改善了了写操作的延时。它会先把写操作发送到后端存储，同时返回给应用写操作完毕，而实际上写的操作还正在执行。使用后写转换器就可以像流水线一样把写请求持续发送。这个后写操作模块更适合使用在client端，以期减少应用的写延迟。

回写中继同样可以聚合写请求。如果aggregate-size选项设置了的话，当连续的写入大小累积起来达到了设定的值，就通过一个写操作写入到存储上。这个操作模式适合应用在服务器端，以为这个可以在多个文件并行被写入磁盘时降低磁头动作。

配置例子：
```bash
volume write-behind-example
type performance/write-behind
option cache-size 3MB    # 缓存大小,当累积达到这个值才进行实际的写操作
option flush-behind on   # 这个参数调整close()/flush()太多的情况，适用于大量小文件的情况
subvolumes
end-volume
```

## 7.9. performance/io-threads (IO线程)

type performance/io-threads

IO线程中继(performance/io-threads）属于性能调整中继的一种，作用是增加IO的并发线程，以提高IO性能。

IO线程中继试图增加服务器后台进程对文件元数据读写I/O的处理能力。由于GlusterFS服务是单线程的，使用IO线程转换器可以较大的提高性能。这个转换器最好是被用于服务器端，而且是在服务器协议转换器后面被加载。

IO线程操作会将读和写操作分成不同的线程。同一时刻存在的总线程是恒定的并且是可以配置的。

配置例子：
```bash
volume iothreads
type performance/io-threads
option thread-count 32 # 线程使用的数量
subvolumes
end-volume
```

## 7.10.    performance/io-cache (IO缓存)

type performance/io-cache

IO缓存中继(performance/io-threads）属于性能调整中继的一种，作用是缓存住已经被读过的数据，以提高IO性能。

IO缓存中继可以缓存住已经被读过的数据。这个对于多个应用对同一个数据多次访问，并且如果读的操作远远大于写的操作的话是很有用的（比如，IO缓存很适合用于提供web服务的环境，大量的客户端只会进行简单的读取文件的操作，只有很少一部分会去写文件）。

当IO缓存中继检测到有写操作的时候，它就会把相应的文件从缓存中删除。

IO缓存中继会定期的根据文件的修改时间来验证缓存中相应文件的一致性。验证超时时间是可以配置的。

配置例子：
```bash
volume iothreads
type performance/ io-cache
option cache-size 32MB  #可以缓存的最大数据量
option cache-timeout 1  #验证超时时间，单位秒
option priority   *:0   #文件匹配列表及其设置的优先级
subvolumes
end-volume
```

## 7.11. 其它中继

其它中继还有
```
cluster/nufa(非均匀文件存取)
cluster/stripe(条带，用于大文件，分块存储在不用服务器)
cluster/ha(集群)
features/filter(过滤)
features/trash(回收站)
path-converter
quota
```
老的还有：
```
cluster/unify（和distribute，可定义不同的调度器，以不同方式写入数据）
```

# 8. GlusterFS配置

## 8.1. 服务器端配置

服务器为6台，IP分别是192.168.1.11,192.168.1.16。配置为：
```bash
# vi /etc/glusterfs/glusterfsd.vol
volume posix
type storage/posix
option directory /sda4
end-volume

volume locks
type features/locks
subvolumes posix
end-volume

volume brick
type performance/io-threads
option thread-count 8
subvolumes locks
end-volume

volume server
type protocol/server
option transport-type tcp
subvolumes brick
option auth.addr.brick.allow *
end-volume
```
保存后启动GlusterFS:
```bash
# service glusterfsd start
```

## 8.2. 客户端配置

服务器为192.168.1.17和192.168.1.18：
```bash
# vi /etc/glusterfs/glusterfs.vol
volume brick1
type protocol/client
option transport-type tcp
end-volume

volume brick2
type protocol/client
option transport-type tcp
option remote-host 192.168.1.12
option remote-subvolume brick
end-volume

volume brick3
type protocol/client
option transport-type tcp
option remote-host 192.168.1.13
option remote-subvolume brick
end-volume

volume brick4
type protocol/client
option transport-type tcp
option remote-host 192.168.1.14
option remote-subvolume brick
end-volume

volume brick5
type protocol/client
option transport-type tcp
option remote-host 192.168.1.15
option remote-subvolume brick
end-volume

volume brick6
type protocol/client
option transport-type tcp
option remote-host 192.168.1.16
option remote-subvolume brick
end-volume

volume afr1
type cluster/replicate
subvolumes brick1 brick2
end-volume

volume afr2
type cluster/replicate
subvolumes brick3 brick4
end-volume

volume afr3
type cluster/replicate
subvolumes brick5 brick6
end-volume

volume unify
type cluster/distribute
subvolumes afr1 afr2 afr3
end-volume
```
GlusterFS的主要配置都在客户端上，上面配置文件的意思是把6台服务器分成3个replicate卷，再用这3个replicate卷做成一个distribute，提供应用程序使用。

## 8.3. GlusterFS挂载

GlusterFS挂载为在客户端上执行：
```bash
# glusterfs -f /etc/glusterfs/glusterfs.vol /gmnt/ -l /var/log/glusterfs/glusterfs.log
```
`-f /etc/glusterfs/glusterfs.vol`为指定GlusterFS的配置文件

`/gmnt`是挂载点

`-l /var/log/glusterfs/glusterfs.log`为日志

另外，GlusterFS也可以结果fstab或autofs方式开机挂载。挂载后就可以在/gmnt内读写文件了，用法与读写本地硬盘一样。


# 9. GlusterFS性能测试

## 9.1. 单客户端测试

测试1：复制大约2.5G容量 /usr目录至GlusterFS(大部分都是小文件)
测试结果：
glusterfs    1361KB/s
本地硬盘   2533KB/s

测试2: 复制一个3.8G的文件至GlusterFS
测试结果：
glusterfs     2270KB/s
本地硬盘    10198KB/s

测试3：读取测试2复制的大文件(cat xxx.iso > /dev/null)
测试结果：
glusterfs     11.2MB/s(基本跑满100M带宽)
本地硬盘    45.6MB/s

## 9.2. 双客户端测试

测试1：在两个客户端上同时复制大约2.5G容量 /usr目录至GlusterFS(大部分都是小文件)
测试结果：
```bash
192.168.1.17:glusterfs   1438KB/s
192.168.1.18:glusterfs   1296KB/s
```

测试2: 在两个客户端上同时复制一个3.8G的文件至GlusterFS
测试结果：
```bash
192.168.1.17:glusterfs    2269KB/s
192.168.1.18:glusterfs    2320KB/s
```

## 9.3. 配置回写功能后的测试

### 9.3.1. 服务器配置
```bash
volume posix
type storage/posix
option directory /sda4
end-volume

volume locks
type features/locks
subvolumes posix
end-volume

volume writebehind
type performance/write-behind
option cache-size   16MB
option flush-behind on
subvolumes locks
end-volume

volume brick
type performance/io-threads
option thread-count 64
subvolumes writebehind
end-volume

volume server
type protocol/server
option transport-type tcp
option auth.addr.brick.allow * # Allow access to “brick” volume
end-volume
```

### 9.3.2. 客户端配置
```bash
volume brick1
type protocol/client
option transport-type tcp
option remote-host 192.168.1.11      # IP address of the remote brick
option remote-subvolume brick        # name of the remote volume
end-volume

volume brick2
type protocol/client
option transport-type tcp
option remote-host 192.168.1.12
option remote-subvolume brick
end-volume

volume brick3
type protocol/client
option transport-type tcp
option remote-host 192.168.1.13
option remote-subvolume brick
end-volume

volume brick4
type protocol/client
option transport-type tcp
option remote-host 192.168.1.14
option remote-subvolume brick
end-volume

volume brick5
type protocol/client
option transport-type tcp
option remote-host 192.168.1.15
option remote-subvolume brick
end-volume

volume brick6
type protocol/client
option transport-type tcp
option remote-host 192.168.1.16
option remote-subvolume brick
end-volume

volume afr1
type cluster/replicate
subvolumes brick1 brick2
end-volume

volume afr2
type cluster/replicate
subvolumes brick3 brick4
end-volume

volume afr3
type cluster/replicate
subvolumes brick5 brick6
end-volume

volume wb1
type performance/write-behind
option cache-size 2MB
option flush-behind on
subvolumes afr1
end-volume

volume wb2
type performance/write-behind
option cache-size 2MB
option flush-behind on
subvolumes afr2
end-volume

volume wb3
type performance/write-behind
option cache-size 2MB
option flush-behind on
subvolumes afr3
end-volume

volume unify
type cluster/distribute
subvolumes wb1 wb2 wb3
end-volume
```

### 9.3.3. 测试

测试：在两个客户端上同时复制大约2.5G容量 /usr目录至GlusterFS(大部分都是小文件)
测试结果：
```
192.168.1.17:glusterfs   979KB/s
192.168.1.18:glusterfs   1056KB/s
```

# 10. 结语

从测试结果看，小文件的写入速度只有1M多，速度过低，好在在多客户端的情况下，写入速度还算平稳。大文件的写入也只有2M。对于做图片服务器来说，只能算勉强够用。

另外在性能调优方面，在我们加上回写后，速度反而有下降。当然也有可能是配置参数不当的原因。

经测试，GlusterFS在高可用方面比较稳定的，基本能达到要求。不过由于在复制模式的更新是通过客户端进行的，当客户端和replicate内的一台服务器同时挂时，会造成数据不同步的情况。需要手动做个列表的动作(ls)才会更新。

GlusterFS作为正式运营环境使用时，还缺乏一些功能，如GlusterFS没有对整个集群的监控和管理程序等。