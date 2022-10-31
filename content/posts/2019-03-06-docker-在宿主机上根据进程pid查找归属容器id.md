---
title: docker 在宿主机上根据进程PID查找归属容器ID
author: 阿辉
date: 2019-03-06T10:17:11+00:00
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
在使用docker时经常出现一台docker主机上跑了多个容器，可能其中一个容器里的进程导致了整个宿主机load很高，其实一条命令就可以找出罪魁祸首

#查找容器ID

`docker inspect -f "{{.Id}} {{.State.Pid}}  {{.Name}} "  $(docker ps -q) |grep <PID>`

#查找k8s pod name

`docker inspect -f "{{.Id}} {{.State.Pid}} {{.Config.Hostname}}"  $(docker ps -q) |grep <PID>`

#如果PID是容器内运行子进程那docker inspect就无法显示了

```bash
for i in  `docker ps |grep Up|awk '{print $1}'`;do echo \ &&docker top $i &&echo ID=$i; done |grep -A 10 <PID>
```
<!--more-->

转自：
https://www.cnblogs.com/37yan/p/9559308.html