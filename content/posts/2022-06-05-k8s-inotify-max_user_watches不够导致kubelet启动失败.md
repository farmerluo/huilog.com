---
title: k8s inotify max_user_watches不够导致kubelet启动失败
author: 阿辉
date: 2022-06-05T09:53:14+00:00
categories:
- Kubernetes
tags:
- Kubernetes
keywords:
- Kubernetes
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
今天升级k8s集群的节点时，发现kubelet启动失败，查看日志发现有如下报错：

```
May 27 16:44:51 sh-saas-k8s1-node-dev-08 kubelet: E0527 16:44:51.876313  453890 watcher.go:152] Failed to watch directory "/sys/fs/cgroup/blkio/kubepods.slice/kubepods-burstable.slice": inotify_add_watch /sys/fs/cgrou
p/blkio/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-podca963772_c0c2_4849_b358_f33363edda4f.slice/docker-b7f3614445d382734d364c74fe612a509f7cb66b1bff6167d53af9ec85d62f0b.scope: no space left on device
May 27 16:44:51 sh-saas-k8s1-node-dev-08 kubelet: E0527 16:44:51.876327  453890 watcher.go:152] Failed to watch directory "/sys/fs/cgroup/blkio/kubepods.slice": inotify_add_watch /sys/fs/cgroup/blkio/kubepods.slice/ku
bepods-burstable.slice/kubepods-burstable-podca963772_c0c2_4849_b358_f33363edda4f.slice/docker-b7f3614445d382734d364c74fe612a509f7cb66b1bff6167d53af9ec85d62f0b.scope: no space left on device
May 27 16:44:51 sh-saas-k8s1-node-dev-08 kubelet: E0527 16:44:51.876371  453890 kubelet.go:1414] "Failed to start cAdvisor" err="inotify_add_watch /sys/fs/cgroup/blkio/kubepods.slice/kubepods-burstable.slice/kubepods-
burstable-podca963772_c0c2_4849_b358_f33363edda4f.slice/docker-b7f3614445d382734d364c74fe612a509f7cb66b1bff6167d53af9ec85d62f0b.scope: no space left on device"
```

google后，发现有人说是inotify用光了，默认值是8192。

```
$ cat /proc/sys/fs/inotify/max_user_watches
8192
```

<!--more-->

可以用如下命令查看当前使用：

```bash
[root@sh-saas-k8s1-node-dev-26 ~]# find /proc/*/fd/ -type l -lname "anon_inode:inotify" -printf "%hinfo/%f\n" | xargs grep -cE "^inotify" | column -t -s:
/proc/1377/fdinfo/6    2
/proc/1406/fdinfo/6    1
/proc/1958/fdinfo/3    2
/proc/1965/fdinfo/5    3
/proc/1/fdinfo/10      1
/proc/1/fdinfo/14      4
/proc/1/fdinfo/15      4
/proc/204666/fdinfo/5  3
/proc/24570/fdinfo/9   1
/proc/24570/fdinfo/26  1
/proc/24570/fdinfo/35  0
/proc/24570/fdinfo/36  3009
/proc/24570/fdinfo/42  1
/proc/525408/fdinfo/3  3
/proc/525408/fdinfo/8  4
/proc/538542/fdinfo/8  0
/proc/807/fdinfo/7     7
/proc/94566/fdinfo/6   60
/proc/94566/fdinfo/7   0
/proc/94566/fdinfo/8   5098
```

可以看看使用量比较高的进程：

```bash
[root@sh-saas-k8s1-node-dev-26 ~]# ps auxw | grep 24570
root       24570 37.0  0.1 2006300 247676 ?      Ssl  16:01  39:37 /usr/local/bin/kubelet --node-status-update-frequency=10s --anonymous-auth=false --authentication-token-webhook --authorization-mode=Webhook --client-ca-file=/etc/kubernetes/ssl/ca.pem --cluster-dns=10.253.252.2 --cluster-domain=cluster.local --cpu-manager-policy=static --network-plugin=kubenet --cloud-provider=external --non-masquerade-cidr=0.0.0.0/0 --cni-bin-dir=/opt/cni/bin --cni-conf-dir=/etc/cni/net.d --pod-infra-container-image=weimob-saas-tcr.hsmob.com/k8s-google-containers/pause-amd64:3.2 --hairpin-mode hairpin-veth --hostname-override=10.12.97.46 --kubeconfig=/etc/kubernetes/kubelet.kubeconfig --max-pods=126 --register-node=true --register-with-taints=install=unfinished:NoSchedule --root-dir=/data/kubernetes/kubelet --tls-cert-file=/etc/kubernetes/ssl/kubelet.pem --tls-private-key-file=/etc/kubernetes/ssl/kubelet-key.pem --cgroups-per-qos=true --cgroup-driver=systemd --enforce-node-allocatable=pods --kube-reserved=cpu=200m,memory=1Gi,ephemeral-storage=1Gi --kube-reserved-cgroup=/system.slice --system-reserved=cpu=200m,memory=1Gi,ephemeral-storage=1Gi --system-reserved-cgroup=/system.slice --eviction-hard=nodefs.available&lt;10%,nodefs.inodesFree&lt;2%,memory.available&lt;1Gi --v=2 --eviction-soft=nodefs.available&lt;15%,nodefs.inodesFree&lt;5%,memory.available&lt;2Gi --eviction-soft-grace-period=memory.available=1m,nodefs.available=1m,nodefs.inodesFree=1m --eviction-max-pod-grace-period=120 --serialize-image-pulls=false --log-dir=/data/log/kubernetes/ --logtostderr=false
root      205305  0.0  0.0 112724  2340 pts/0    S+   17:48   0:00 grep --color=auto 24570

[root@sh-saas-k8s1-node-dev-26 ~]# ps auxw | grep 94566
root       94566  1.5  0.1 1209712 164404 ?      Sl   16:37   1:07 /opt/conda/bin/python3 /app/webot3-robot3/manage.py runserver -h 0.0.0.0 -p 8080
root      205768  0.0  0.0 112724  2264 pts/0    S+   17:48   0:00 grep --color=auto 94566
[root@sh-saas-k8s1-node-dev-26 ~]#
```

问题解决也很简单，调大一下就好了。

```bash
[root@sh-saas-k8s1-master-dev-01 ansible]# sysctl -w fs.inotify.max_user_watches=81920
fs.inotify.max_user_watches = 81920
[root@sh-saas-k8s1-master-dev-01 ansible]# echo 'fs.inotify.max_user_watches=81920' >> /etc/sysctl.conf
```

打开 inotify\_add\_watch 跟踪，进一步 debug inotify watch 耗尽的原因:

```
echo 1 > /sys/kernel/debug/tracing/events/syscalls/sys_exit_inotify_add_watch/enable
```