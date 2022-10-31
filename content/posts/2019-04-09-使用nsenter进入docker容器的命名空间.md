---
title: 使用nsenter进入docker容器的命名空间
author: 阿辉
date: 2019-04-09T05:59:32+00:00
categories:
- Kubernetes
- Docker
tags:
- Kubernetes
- Docker
keywords:
- Kubernetes
- Docker
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

centos 7 已经自喧nsenter这个命令，可以直接使用，它可以方便的让我们进入docker容器的命名空间。

首先获取容器pid，示例如下：

```bash
[root@sh-saas-k8s1-master-dev-01 ~]# docker ps
CONTAINER ID        IMAGE                                                                 COMMAND                  CREATED             STATUS              PORTS               NAMES
f8b1e0b8caa7        nginx                                                                 "nginx -g 'daemon of…"   33 seconds ago      Up 33 seconds       80/tcp              nginx
[root@sh-saas-k8s1-master-dev-01 ~]# pid=$(docker inspect --format "{{ .State.Pid }}" f8b1e0b8caa7)
[root@sh-saas-k8s1-master-dev-01 ~]# echo $pid
16042
```

然后使用nsenter命令进入：
```bash
[root@sh-saas-k8s1-master-dev-01 ~]# nsenter --target $pid --mount --uts --ipc --net --pid
mesg: ttyname failed: No such file or directory
root@f8b1e0b8caa7:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@f8b1e0b8caa7:/# ip a
-bash: ip: command not found
root@f8b1e0b8caa7:/# exit
logout
```
<!--more-->

我们看到报了一个错，但是还是进去了，可以指定一个执行的shell：
```bash
[root@sh-saas-k8s1-master-dev-01 ~]# nsenter --target $pid --mount --uts --ipc --net --pid /bin/bash
root@f8b1e0b8caa7:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```
这样就不会报错了。

也可以只进入网络命名空间，这样可以利用主机上的命令来测试：
```bash
[root@sh-saas-k8s1-master-dev-01 ~]# nsenter --target $pid --net ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
41: eth0@if42: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
可以看到，在之前测试时，容器内是没有ip 这个命令的，现在只挂网络命名空间后，就可以通过主机上的ip命令执行了。而执行的结果是容器内的。