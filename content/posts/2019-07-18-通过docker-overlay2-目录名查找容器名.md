---
title: 通过docker overlay2 目录名查找容器名
author: 阿辉
date: 2019-07-18T06:50:54+00:00
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

有时候经常会有个别容器占用磁盘空间特别大，这个时候就需要通过docker overlay2 目录名查找容器名：

先进入overlay2的目录，这个目录可以通过docker的配置文件（/etc/docker/daemon.json）内找到。然后看看谁占用空间比较多。
```bash
[root@sh-saas-k8s1-node-qa-04 overlay2]# du -sc * | sort -rn  | more
33109420        total
1138888 20049e2e445181fc742b9e74a8819edf0e7ee8f0c0041fb2d1c9d321f73d8f5b
1066548 010d0a26a1fe5b00e330d0d87649fc73af45f9333fd824bf0f9d91a37276af18
943208  030c0f111675f6ed534eaa6e4183ec91d4c065dd4bdb5a289f4b572357667378
825116  0ad9e737795dd367bb72f7735fb69a65db3d8907305b305ec21232505241d044
824756  bf3c698966bc19318f3263631bc285bde07c6a1a4eaea25c4ecd3b7b8f29b3fd
661000  15763b72802e1e71cc943e09cba8b747779bf80fa35d56318cf1b89f7b1f1e71
575564  02eaa52e2f999dc387a9dee543028bada0762022cef1400596b5cc18a6223635
486780  4353c30611d7f51932d9af24bb1330db0fdb86faa9d9cae02ed618fa975c697a
486420  562a8874cc345b8ea830c1486c42211b288c886c5dca08e14d7057cacab984c1
486420  4f897e8cd355320b0e7ee1ecc9db5c43d5151f9afa29f1925fe264c88429be4c
448652  a8d0596d123fcc59983ce63df3f3acd40d5c930ed72874ce0a9efbc3234466de
448296  851cc4912edb9989e120acf241f26c82e9715e7fcb1d2bd19a002fcfb894f1f4
417780  20608baacae6bafcd4230a18a272857bc75703a6eefef5c9b40ba4ea19496b11
387388  43a8a76de3b5531e4c12f956f7bfcfcdb8fd38548faf20812cafa9a39813abc5
```

<!--more-->

再通过目录名查找容器名：
```bash
[root@sh-saas-k8s1-node-qa-04 overlay2]#  docker ps -q | xargs docker inspect --format '{{.State.Pid}}, {{.Name}}, {{.GraphDriver.Data.WorkDir}}' | grep "20049e2e445181fc742b9e74a8819edf0e7ee8f0c0041fb2d1c9d321f73d8f5b"
4884, /k8s_taskmanager_flink-taskmanager-java-qa-7879d55f45-xbd74_public-tjpm-flink-java-qa_ad8bf915-a23f-11e9-be66-52540088db9a_0, /data/kubernetes/docker/overlay2/20049e2e445181fc742b9e74a8819edf0e7ee8f0c0041fb2d1c9d321f73d8f5b/work
```

如果发现有目录查不到，通常是因为容器已经被删掉了，目录没有清理，这时直接清理便可：
```bash
docker system prune -a -f
```