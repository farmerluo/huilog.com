---
title: kubelet 启动失败failed to build map of initial containers from runtime no Podsand
author: 阿辉
date: 2022-06-05T09:54:36+00:00
categories:
- kubernetes
tags:
- kubernetes
keywords:
- kubernetes
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

kubelet 启动失败，报错为：failed to build map of initial containers from runtime: no PodsandBox found with Id &#8216;d08297f6a1a3c25c88c6155005778e36be90b8919383c88dfe22b7313a984d89&#8217;

详细如下：

```bash
May 31 19:36:53 sh-saas-k8s1-node-qa-52 kubelet: Flag --logtostderr has been deprecated, will be removed in a future release, see https://github.com/kubernetes/enhancements/tree/master/keps/sig-instrumentation/2845-deprecate-klog-specific-flags-in-k8s-components
May 31 19:36:53 sh-saas-k8s1-node-qa-52 systemd: Started Kubernetes systemd probe.
May 31 19:36:53 sh-saas-k8s1-node-qa-52 kubelet: E0531 19:36:53.648850   87963 kubelet.go:1351] "Image garbage collection failed once. Stats initialization may not have completed yet" err="failed to get imageFs info: unable to find data in memory cache"
May 31 19:36:53 sh-saas-k8s1-node-qa-52 kubelet: E0531 19:36:53.657160   87963 kubelet.go:2386] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: kubenet does not have netConfig. This is most likely due to lack of PodCIDR"
May 31 19:36:53 sh-saas-k8s1-node-qa-52 kubelet: E0531 19:36:53.675950   87963 kubelet.go:2040] "Skipping pod synchronization" err="[container runtime status check may not have completed yet, PLEG is not healthy: pleg has yet to be successful]"
May 31 19:36:53 sh-saas-k8s1-node-qa-52 kubelet: E0531 19:36:53.743460   87963 kubelet.go:1431] "Failed to start ContainerManager" err="failed to build map of initial containers from runtime: no PodsandBox found with Id 'd08297f6a1a3c25c88c6155005778e36be90b8919383c88dfe22b7313a984d89'"
```

主要是找不到runtime内对应ID的容器，id为&#8217;d08297f6a1a3c25c88c6155005778e36be90b8919383c88dfe22b7313a984d89&#8242;

<!--more-->

解决办法：

通过下面的命令找到容器id:

```bash
docker ps -a --filter "label=io.kubernetes.sandbox.id=d08297f6a1a3c25c88c6155005778e36be90b8919383c88dfe22b7313a984d89"
```

再删除容器：

```bash
docker rm ID
```

如果找不到容器ID了，可以：  
1. 踢掉节点,停掉kubelet: `systemctl stop kubelet`  
2. 删除所有容器: `docker rm $(docker ps -a -q)`  
3. 使用`docker system prune -a -f  --volumes`清理容器  
4. 重启kubelet

参考：

  * https://github.com/kubernetes/kubelet/issues/21
  * https://serverfault.com/questions/1081253/kubernetes-failing-to-start-failed-to-build-map-of-initial-containers
  * https://github.com/kubernetes/kubernetes/pull/98225