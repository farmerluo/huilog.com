---
title: kube-proxy监控发现规则同步时高延迟现象排查
author: 阿辉
date: 2022-12-22T22:53:00+08:00
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
{{< toc >}}

# 1. kube proxy规则同步高延迟现象

在一次重启coredns时，我们发现有些解析失败的现象，由此怀疑到可能是kube proxy更新ipvs或iptables规则慢导致的。

查看kube proxy的监控图，也确实发现有偶尔非常慢的问题，如下图所示：

![kubeproxy1](/static/images/2022/kube-proxy1.png)

可以看到10.11.96.92这个节点显示规则同步的99线为16.4s。如果这个时间是真的，那肯定是不能接受的一个延迟时间。正常来说应该不应该超过1s。

<!--more-->

# 2. kube proxy源码分析及其实现原理

通过对kube proxy的源码分析来了解其实现原理，看看kube proxy是怎么更新ipvs规则的？

1. kube proxy会watch service、endpoints和endpointSlices这三种对象，并实现了 OnAdd、OnUpdate、OnDelete和OnSynced方法：

kube proxy的核心代码基本都在pkg/proxy/ipvs/proxier.go：
```golang
// OnServiceAdd is called whenever creation of new service object is observed.
func (proxier *Proxier) OnServiceAdd(service *v1.Service) {
	proxier.OnServiceUpdate(nil, service)
}

// OnServiceUpdate is called whenever modification of an existing service object is observed.
func (proxier *Proxier) OnServiceUpdate(oldService, service *v1.Service) {
	if proxier.serviceChanges.Update(oldService, service) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnServiceDelete is called whenever deletion of an existing service object is observed.
func (proxier *Proxier) OnServiceDelete(service *v1.Service) {
	proxier.OnServiceUpdate(service, nil)
}

// OnServiceSynced is called once all the initial event handlers were called and the state is fully propagated to local cache.
func (proxier *Proxier) OnServiceSynced() {
	proxier.mu.Lock()
	proxier.servicesSynced = true
	proxier.setInitialized(proxier.endpointSlicesSynced)
	proxier.mu.Unlock()

	// Sync unconditionally - this is called once per lifetime.
	proxier.syncProxyRules()
}

// OnEndpointSliceAdd is called whenever creation of a new endpoint slice object
// is observed.
func (proxier *Proxier) OnEndpointSliceAdd(endpointSlice *discovery.EndpointSlice) {
	if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, false) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnEndpointSliceUpdate is called whenever modification of an existing endpoint
// slice object is observed.
func (proxier *Proxier) OnEndpointSliceUpdate(_, endpointSlice *discovery.EndpointSlice) {
	if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, false) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnEndpointSliceDelete is called whenever deletion of an existing endpoint slice
// object is observed.
func (proxier *Proxier) OnEndpointSliceDelete(endpointSlice *discovery.EndpointSlice) {
	if proxier.endpointsChanges.EndpointSliceUpdate(endpointSlice, true) && proxier.isInitialized() {
		proxier.Sync()
	}
}

// OnEndpointSlicesSynced is called once all the initial event handlers were
// called and the state is fully propagated to local cache.
func (proxier *Proxier) OnEndpointSlicesSynced() {
	proxier.mu.Lock()
	proxier.endpointSlicesSynced = true
	proxier.setInitialized(proxier.servicesSynced)
	proxier.mu.Unlock()

	// Sync unconditionally - this is called once per lifetime.
	proxier.syncProxyRules()
}

// OnNodeAdd is called whenever creation of new node object
// is observed.
func (proxier *Proxier) OnNodeAdd(node *v1.Node) {
	if node.Name != proxier.hostname {
		klog.ErrorS(nil, "Received a watch event for a node that doesn't match the current node", "eventNode", node.Name, "currentNode", proxier.hostname)
		return
	}

	if reflect.DeepEqual(proxier.nodeLabels, node.Labels) {
		return
	}

	proxier.mu.Lock()
	proxier.nodeLabels = map[string]string{}
	for k, v := range node.Labels {
		proxier.nodeLabels[k] = v
	}
	proxier.mu.Unlock()
	klog.V(4).InfoS("Updated proxier node labels", "labels", node.Labels)

	proxier.syncProxyRules()
}

// OnNodeUpdate is called whenever modification of an existing
// node object is observed.
func (proxier *Proxier) OnNodeUpdate(oldNode, node *v1.Node) {
	if node.Name != proxier.hostname {
		klog.ErrorS(nil, "Received a watch event for a node that doesn't match the current node", "eventNode", node.Name, "currentNode", proxier.hostname)
		return
	}

	if reflect.DeepEqual(proxier.nodeLabels, node.Labels) {
		return
	}

	proxier.mu.Lock()
	proxier.nodeLabels = map[string]string{}
	for k, v := range node.Labels {
		proxier.nodeLabels[k] = v
	}
	proxier.mu.Unlock()
	klog.V(4).InfoS("Updated proxier node labels", "labels", node.Labels)

	proxier.syncProxyRules()
}

// OnNodeDelete is called whenever deletion of an existing node
// object is observed.
func (proxier *Proxier) OnNodeDelete(node *v1.Node) {
	if node.Name != proxier.hostname {
		klog.ErrorS(nil, "Received a watch event for a node that doesn't match the current node", "eventNode", node.Name, "currentNode", proxier.hostname)
		return
	}
	proxier.mu.Lock()
	proxier.nodeLabels = nil
	proxier.mu.Unlock()

	proxier.syncProxyRules()
}

// OnNodeSynced is called once all the initial event handlers were
// called and the state is fully propagated to local cache.
func (proxier *Proxier) OnNodeSynced() {
}
```

这些方法最终都是调用proxier.syncProxyRules()或proxier.Sync()，而proxier.Sync()最终也还是会执行proxier.syncProxyRules()来同步ipvs规则。

2. proxier.syncRunner() 执行流程：

- 拉取service及endpoint列表，注意service及endpoint的更新是异步的：

```golang
// This is where all of the ipvs calls happen.
func (proxier *Proxier) syncProxyRules() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()

	// don't sync rules till we've received services and endpoints
	if !proxier.isInitialized() {
		klog.V(2).InfoS("Not syncing ipvs rules until Services and Endpoints have been received from master")
		return
	}

	// Keep track of how long syncs take.
	start := time.Now()
	defer func() {
		metrics.SyncProxyRulesLatency.Observe(metrics.SinceInSeconds(start))
		klog.V(4).InfoS("syncProxyRules complete", "elapsed", time.Since(start))
	}()

	// We assume that if this was called, we really want to sync them,
	// even if nothing changed in the meantime. In other words, callers are
	// responsible for detecting no-op changes and not calling this function.
	serviceUpdateResult := proxier.serviceMap.Update(proxier.serviceChanges)
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)

	staleServices := serviceUpdateResult.UDPStaleClusterIP
	// merge stale services gathered from updateEndpointsMap
	for _, svcPortName := range endpointUpdateResult.StaleServiceNames {
		...
	}

	klog.V(3).InfoS("Syncing ipvs proxier rules")
    ...
```

- 通过iptables-save获取现有的Filter和NAT表存在的链数据,创建dummy interface kube-ipvs0，创建默认的ipset 规则。
```golang
	// Begin install iptables

	// Reset all buffers used later.
	// This is to avoid memory reallocations and thus improve performance.
	proxier.natChains.Reset()
	proxier.natRules.Reset()
	proxier.filterChains.Reset()
	proxier.filterRules.Reset()

	// Write table headers.
	proxier.filterChains.Write("*filter")
	proxier.natChains.Write("*nat")

	proxier.createAndLinkKubeChain()
	
	// 创建dummy interface kube-ipvs0
	_, err := proxier.netlinkHandle.EnsureDummyDevice(DefaultDummyDevice)
	if err != nil {
		klog.ErrorS(err, "Failed to create dummy interface", "interface", DefaultDummyDevice)
		return
	}

	// 创建默认的ipset 规则
	for _, set := range proxier.ipsetList {
		if err := ensureIPSet(set); err != nil {
			return
		}
		set.resetEntries()
	}
	...
```

- 对serviceMap内的每个服务进行遍历处理，针对不同的服务类型(clusterip/nodePort/externalIPs/load-balancer)进行不同的处理(ipset列表/vip/ipvs后端服务器)

根据 endpoint 列表，更新 KUBE-LOOP-BACK 的 ipset 列表

若为 clusterIP 类型更新对应的 ipset 列表 KUBE-CLUSTER-IP

若为 externalIPs 类型更新对应的 ipset 列表 KUBE-EXTERNAL-IP

若为 load-balancer 类型更新对应的 ipset 列表KUBE-LOAD-BALANCER、KUBE-LOAD-BALANCER-LOCAL、KUBE-LOAD-BALANCER-FW、KUBE-LOAD-BALANCER-SOURCE-CIDR、KUBE-LOAD-BALANCER-SOURCE-IP

若为 NodePort 类型更新对应的 ipset 列表 KUBE-NODE-PORT-TCP、KUBE-NODE-PORT-LOCAL-TCP、KUBE-NODE-PORT-LOCAL-SCTP-HASH、KUBE-NODE-PORT-LOCAL-UDP、KUBE-NODE-PORT-SCTP-HASH、KUBE-NODE-PORT-UDP

我们只看clusterIP这一块的代码：
```golang
	// Build IPVS rules for each service.
	for svcName, svc := range proxier.serviceMap {
		svcInfo, ok := svc.(*serviceInfo)
		if !ok {
			klog.ErrorS(nil, "Failed to cast serviceInfo", "serviceName", svcName)
			continue
		}
		isIPv6 := netutils.IsIPv6(svcInfo.ClusterIP())
		localPortIPFamily := netutils.IPv4
		if isIPv6 {
			localPortIPFamily = netutils.IPv6
		}
		protocol := strings.ToLower(string(svcInfo.Protocol()))
		// Precompute svcNameString; with many services the many calls
		// to ServicePortName.String() show up in CPU profiles.
		svcNameString := svcName.String()

		// Handle traffic that loops back to the originator with SNAT.
		for _, e := range proxier.endpointsMap[svcName] {
			...
			proxier.ipsetList[kubeLoopBackIPSet].activeEntries.Insert(entry.String())
		}

		// Capture the clusterIP.
		// ipset call
		entry := &utilipset.Entry{
			IP:       svcInfo.ClusterIP().String(),
			Port:     svcInfo.Port(),
			Protocol: protocol,
			SetType:  utilipset.HashIPPort,
		}
		// add service Cluster IP:Port to kubeServiceAccess ip set for the purpose of solving hairpin.
		// proxier.kubeServiceAccessSet.activeEntries.Insert(entry.String())
		if valid := proxier.ipsetList[kubeClusterIPSet].validateEntry(entry); !valid {
			klog.ErrorS(nil, "Error adding entry to ipset", "entry", entry, "ipset", proxier.ipsetList[kubeClusterIPSet].Name)
			continue
		}
		proxier.ipsetList[kubeClusterIPSet].activeEntries.Insert(entry.String())
		// ipvs call
		serv := &utilipvs.VirtualServer{
			Address:   svcInfo.ClusterIP(),
			Port:      uint16(svcInfo.Port()),
			Protocol:  string(svcInfo.Protocol()),
			Scheduler: proxier.ipvsScheduler,
		}
		// Set session affinity flag and timeout for IPVS service
		if svcInfo.SessionAffinityType() == v1.ServiceAffinityClientIP {
			serv.Flags |= utilipvs.FlagPersistent
			serv.Timeout = uint32(svcInfo.StickyMaxAgeSeconds())
		}
		// We need to bind ClusterIP to dummy interface, so set `bindAddr` parameter to `true` in syncService()
		if err := proxier.syncService(svcNameString, serv, true, bindedAddresses); err == nil {
			activeIPVSServices[serv.String()] = true
			activeBindAddrs[serv.Address.String()] = true
			// ExternalTrafficPolicy only works for NodePort and external LB traffic, does not affect ClusterIP
			// So we still need clusterIP rules in onlyNodeLocalEndpoints mode.
			internalNodeLocal := false
			if utilfeature.DefaultFeatureGate.Enabled(features.ServiceInternalTrafficPolicy) && svcInfo.NodeLocalInternal() {
				internalNodeLocal = true
			}
			if err := proxier.syncEndpoint(svcName, internalNodeLocal, serv); err != nil {
				klog.ErrorS(err, "Failed to sync endpoint for service", "serviceName", svcName, "virtualServer", serv)
			}
		} else {
			klog.ErrorS(err, "Failed to sync service", "serviceName", svcName, "virtualServer", serv)
		}
		...

```
- 最后同步 ipset 记录,刷新 iptables 规则等
```golang
    ...
	// sync ipset entries
	for _, set := range proxier.ipsetList {
		set.syncIPSetEntries()
	}

	// Tail call iptables rules for ipset, make sure only call iptables once
	// in a single loop per ip set.
	proxier.writeIptablesRules()

	// Sync iptables rules.
	// NOTE: NoFlushTables is used so we don't flush non-kubernetes chains in the table.
	proxier.iptablesData.Reset()
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
	proxier.iptablesData.Write(proxier.filterChains.Bytes())
	proxier.iptablesData.Write(proxier.filterRules.Bytes())

	klog.V(5).InfoS("Restoring iptables", "rules", string(proxier.iptablesData.Bytes()))
	err = proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)
	if err != nil {
		klog.ErrorS(err, "Failed to execute iptables-restore", "rules", string(proxier.iptablesData.Bytes()))
		metrics.IptablesRestoreFailuresTotal.Inc()
		// Revert new local ports.
		utilproxy.RevertPorts(replacementPortsMap, proxier.portsMap)
		return
	}
	...
```

# 3. kube proxy规则同步高延迟初步排查

从以前对ipvs这块的测试数据及使用经验来看，ipvs规则的更新通常是比较快的。

因此分析以上问题可能有2个原因：
- 是否监控数据有些问题？
- ipvs的规则更新因某些原因真的慢了

根据前面对kube proxy的源码分析，在`proxier.syncRunner()`内`klog.V(4).InfoS("syncProxyRules complete", "elapsed", time.Since(start))`这行记录了syncProxyRules执行完成所需要的时间。可以把这行日志打印出来，当从监控上发现延迟很长情况时，找到这条日志来对比验证。

在分析日志时从日志：`Syncing ipvs proxier rules`开始看，这是执行规则同步的起点。

下面修改kube-proxy的启动参数，增加`--v=5`:
```console
# /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/data/kubernetes/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --hostname-override=10.11.96.100 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --log-dir=/data/log/kubernetes/ \
  --logtostderr=false \
  --v=5 \
  --proxy-mode=ipvs \
  --metrics-bind-address=0.0.0.0:10249
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

重启kube-proxy:`systemctl restart kube-proxy`，让其生效。后续就是观察监控，看到延迟高时就到相应的节点上查看kube proxy的日志。

过了一段时间后，发现有两台节点显示有16s的延迟：

![kubeproxy2](/static/images/2022/kube-proxy2.png)

由上面的监控图可以发现:
- 10.11.96.100这台节点在18:45到18:50延迟14s.
- 10.11.96.111这台节点在19:06延迟16s.

下面去查看这两台节点查看kube proxy的日志

10.11.96.100节点的kube proxy日志:

```console
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.INFO | grep "syncProxyRules" | grep "I1214 18:4" | grep -v ms
I1214 18:44:27.546600   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.442771651s"
I1214 18:44:42.571976   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.025248788s"
```

10.11.96.100这台节点syncProxyRules执行时间确实有2次15s的，发现正是kube proxy启动后的前2次规则同步，后面没有慢的现象。

10.11.96.111节点的kube proxy日志:
```console
[root@sh-saas-k8s1-node-qa-111 kubernetes]# cat kube-proxy.INFO | grep "syncProxyRules" | grep -v ms
[root@sh-saas-k8s1-node-qa-111 kubernetes]#
```

10.11.96.111这台节点，均没有执行时间超过1s的情况。

我们再看其它节点的情况：
```console
[root@sh-saas-k8s1-master-qa-01 ansible-cni]# ansible kube-node -m shell -a "cat /data/log/kubernetes/kube-proxy.INFO | grep 'syncProxyRules' | grep -v ms"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
10.11.96.104 | FAILED | rc=1 >>
non-zero return code
10.11.96.108 | FAILED | rc=1 >>
non-zero return code
10.11.96.97 | FAILED | rc=1 >>
non-zero return code
10.11.96.109 | CHANGED | rc=0 >>
I1214 18:35:36.754652  806918 proxier.go:1008] "syncProxyRules complete" elapsed="1.742004105s"
I1214 18:35:38.365159  806918 proxier.go:1008] "syncProxyRules complete" elapsed="1.610400634s"
10.11.96.96 | CHANGED | rc=0 >>
I1214 18:30:20.074185 1047667 proxier.go:1008] "syncProxyRules complete" elapsed="6.291050508s"
I1214 18:30:26.336030 1047667 proxier.go:1008] "syncProxyRules complete" elapsed="6.261717583s"
10.11.96.103 | FAILED | rc=1 >>
non-zero return code
10.11.96.102 | FAILED | rc=1 >>
non-zero return code
10.11.96.99 | FAILED | rc=1 >>
non-zero return code
10.11.96.110 | CHANGED | rc=0 >>
I1219 19:00:12.625572  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.00417046s"
I1219 19:00:53.623217  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.058888462s"
I1219 19:01:17.093380  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.050323555s"
I1219 19:01:22.909836  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.352718929s"
I1219 19:01:27.091561  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.077237668s"
10.11.96.105 | CHANGED | rc=0 >>
I1214 18:40:29.298876  540671 proxier.go:1008] "syncProxyRules complete" elapsed="2.809855386s"
I1214 18:40:31.870117  540671 proxier.go:1008] "syncProxyRules complete" elapsed="2.57112435s"
10.11.96.107 | CHANGED | rc=0 >>
I1214 18:37:42.275860  662350 proxier.go:1008] "syncProxyRules complete" elapsed="2.678461138s"
I1214 18:37:44.768592  662350 proxier.go:1008] "syncProxyRules complete" elapsed="2.492609698s"
10.11.96.106 | CHANGED | rc=0 >>
I1214 18:39:14.135275  502232 proxier.go:1008] "syncProxyRules complete" elapsed="3.368378997s"
I1214 18:39:17.486706  502232 proxier.go:1008] "syncProxyRules complete" elapsed="3.351377599s"
10.11.96.100 | CHANGED | rc=0 >>
I1214 18:44:27.546600   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.442771651s"
I1214 18:44:42.571976   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.025248788s"
10.11.96.95 | CHANGED | rc=0 >>
I1214 18:30:58.095665 1011586 proxier.go:1008] "syncProxyRules complete" elapsed="5.418281208s"
I1214 18:31:03.603491 1011586 proxier.go:1008] "syncProxyRules complete" elapsed="5.507788493s"
I1219 18:00:06.599770 1011586 proxier.go:1008] "syncProxyRules complete" elapsed="1.007590745s"
10.11.96.98 | CHANGED | rc=0 >>
I1214 19:27:44.586760  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.015047146s"
I1215 15:02:33.378674  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.306158272s"
I1215 17:51:58.651549  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.090296314s"
I1215 18:55:27.000296  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.169267291s"
I1215 18:55:28.319427  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.319091838s"
I1215 19:01:18.928222  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.147711822s"
I1216 14:41:18.297584  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.196060478s"
I1216 14:52:36.661525  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.110995506s"
I1216 14:52:37.987179  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.325606522s"
I1216 15:38:09.018547  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.405135864s"
I1216 16:05:06.535796  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.425388635s"
I1219 15:06:22.626687  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.131007056s"
10.11.96.111 | FAILED | rc=1 >>
non-zero return code
10.11.96.113 | FAILED | rc=1 >>
non-zero return code
10.11.96.115 | FAILED | rc=1 >>
non-zero return code
10.11.96.112 | FAILED | rc=1 >>
non-zero return code
10.11.96.114 | FAILED | rc=1 >>
non-zero return code
```

一共20台节点，其中有11台节点没有超过1s的延迟情况，9台存在超过1s的延迟情况。
```
[root@sh-saas-k8s1-master-qa-01 ansible-cni]# ansible kube-node -m shell -a "cat /data/log/kubernetes/kube-proxy.INFO | grep 'Using ipvs Proxier'"
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
10.11.96.104 | CHANGED | rc=0 >>
I0902 16:24:11.082227   29761 server_others.go:269] "Using ipvs Proxier"
10.11.96.97 | CHANGED | rc=0 >>
I1214 18:29:12.118633  941200 server_others.go:269] "Using ipvs Proxier"
10.11.96.99 | CHANGED | rc=0 >>
I1214 15:50:22.964830  934493 server_others.go:269] "Using ipvs Proxier"
10.11.96.96 | CHANGED | rc=0 >>
I1214 18:30:13.577270 1047667 server_others.go:269] "Using ipvs Proxier"
10.11.96.108 | CHANGED | rc=0 >>
I1214 18:36:11.719670  171957 server_others.go:269] "Using ipvs Proxier"
10.11.96.100 | CHANGED | rc=0 >>
I1214 18:44:11.836130   98073 server_others.go:269] "Using ipvs Proxier"
10.11.96.103 | CHANGED | rc=0 >>
I1214 18:42:55.509473  569213 server_others.go:269] "Using ipvs Proxier"
10.11.96.102 | CHANGED | rc=0 >>
I1214 18:43:33.420310  246832 server_others.go:269] "Using ipvs Proxier"
10.11.96.109 | CHANGED | rc=0 >>
I1214 18:35:34.807031  806918 server_others.go:269] "Using ipvs Proxier"
10.11.96.95 | CHANGED | rc=0 >>
I1214 18:30:52.473627 1011586 server_others.go:269] "Using ipvs Proxier"
10.11.96.110 | CHANGED | rc=0 >>
I1214 18:34:41.881756  455291 server_others.go:269] "Using ipvs Proxier"
10.11.96.106 | CHANGED | rc=0 >>
I1214 18:39:10.561900  502232 server_others.go:269] "Using ipvs Proxier"
10.11.96.107 | CHANGED | rc=0 >>
I1214 18:37:39.392868  662350 server_others.go:269] "Using ipvs Proxier"
10.11.96.105 | CHANGED | rc=0 >>
I1214 18:40:26.283641  540671 server_others.go:269] "Using ipvs Proxier"
10.11.96.98 | CHANGED | rc=0 >>
I1214 18:26:59.562197  250794 server_others.go:269] "Using ipvs Proxier"
10.11.96.111 | CHANGED | rc=0 >>
I1214 18:34:04.093513  614210 server_others.go:269] "Using ipvs Proxier"
10.11.96.112 | CHANGED | rc=0 >>
I1214 18:33:28.220148  674855 server_others.go:269] "Using ipvs Proxier"
10.11.96.113 | CHANGED | rc=0 >>
I1214 18:32:50.924190  711113 server_others.go:269] "Using ipvs Proxier"
10.11.96.114 | CHANGED | rc=0 >>
I1214 18:32:13.936128  647611 server_others.go:269] "Using ipvs Proxier"
10.11.96.115 | CHANGED | rc=0 >>
I1214 18:31:33.524141  634902 server_others.go:269] "Using ipvs Proxier"
```

先看看有没有什么规律：
- kube proxy启动时前2次规则同步延迟高的节点：10.11.96.109，10.11.96.96，10.11.96.100，10.11.96.105，10.11.96.106，10.11.96.107，这几台节点的规律是启动时有2次规则同步延迟高，后面均没有再出现。
- 10.11.96.95这台节点除了启动时有2次规则同步延迟高，后面也出现了一次规则同步的高延迟。时间为1s多一点点
- 10.11.96.110及10.11.96.98为启动时正常，启动后出现了规则同步延迟高的情况，延迟时间均为1~2s
- 结合上面的监控图来看，像10.11.96.111监控显示有超过10s的延迟，实际上在日志上看不到有超过1s的延迟

现在对这些节点的延迟我们大概分下类：
- kube proxy启动时的前2次规则同步高延迟，延迟时间为1～16s。节点有：10.11.96.109，10.11.96.96，10.11.96.100，10.11.96.105，10.11.96.106，10.11.96.107，10.11.96.95。
- kube proxy正常运行时的规则同步高延迟，延迟时间为1～2s。节点有：10.11.96.110，10.11.96.98，10.11.96.95
- 监控显示延迟高，实际日志上看不高，节点最少有：10.11.96.111

后面针对这几类延迟再一个一个进行分析排查。

# 4. kube proxy规则同步高延迟原因分析

## 4.1 kube proxy启动时前2次规则同步的高延迟原因分析

从10.11.96.100节点开始查，其前2次规则同步高延迟的时间为:

```console
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.INFO | grep "syncProxyRules" | grep "I1214 18:4" | grep -v ms
I1214 18:44:27.546600   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.442771651s"
I1214 18:44:42.571976   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.025248788s"
```

先看`I1214 18:44:27.546600   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.442771651s"`的这次延迟，相关的日志如下：
```console
I1214 18:44:12.168986   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:12.168992   98073 iptables.go:357] running iptables-save [-t filter]
I1214 18:44:12.164906   98073 service.go:304] "Service updated ports" service="saas-caiwu-tomcat-qa/saas-caiwu-wlc-logistics-web-tomcat-qa" portCount=1
I1214 18:44:12.169119   98073 config.go:336] "Calling handler.OnServiceAdd"
I1214 18:44:12.169142   98073 service.go:304] "Service updated ports" service="ep-java-qa/multienvtest10-java-qa-standard-big8" portCount=1
...
I1214 18:44:12.203594   98073 service.go:304] "Service updated ports" service="o2o-java-qa/nco-manager-api-java-qa-standard" portCount=1
I1214 18:44:12.203599   98073 config.go:336] "Calling handler.OnServiceAdd"
I1214 18:44:12.203608   98073 service.go:304] "Service updated ports" service="public-tjpm-node-nginx-qa/public-tjpm-albus-web-node-nginx-qa" portCount=1
I1214 18:44:12.331507   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.171.6:443/TCP/10.252.30.86:9443"
I1214 18:44:12.331525   98073 graceful_termination.go:159] "Trying to delete real server" realServer="10.252.171.6:443/TCP/10.252.30.86:9443"
I1214 18:44:12.331548   98073 graceful_termination.go:173] "Deleting real server" realServer="10.252.171.6:443/TCP/10.252.30.86:9443"
I1214 18:44:12.331619   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.207.182:8080/TCP/10.252.18.152:8080"
...
I1214 18:44:12.344390   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.252.1:443/TCP/10.11.96.59:8443"
I1214 18:44:12.344396   98073 graceful_termination.go:159] "Trying to delete real server" realServer="10.252.252.1:443/TCP/10.11.96.59:8443"
I1214 18:44:12.344421   98073 graceful_termination.go:170] "Skip deleting real server till all connection have expired" realServer="10.252.252.1:443/TCP/10.11.96.59:8443" activeConnection=4 inactiveConnection=0
I1214 18:44:12.344441   98073 graceful_termination.go:153] "Adding real server to graceful delete real server list" realServer="10.252.252.1:443/TCP/10.11.96.59:8443"
I1214 18:44:12.344449   98073 graceful_termination.go:65] "Adding real server to graceful delete real server list" realServer="10.252.252.1:443/TCP/10.11.96.59:8443"
I1214 18:44:12.344780   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.169.192:8080/TCP/10.252.23.89:8080"
...
I1214 18:44:12.395570   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.184.54:8009/TCP/10.252.24.117:8009"
I1214 18:44:12.395576   98073 graceful_termination.go:159] "Trying to delete real server" realServer="10.252.184.54:8009/TCP/10.252.24.117:8009"
I1214 18:44:12.395597   98073 graceful_termination.go:173] "Deleting real server" realServer="10.252.184.54:8009/TCP/10.252.24.117:8009"
I1214 18:44:12.395807   98073 proxier.go:1453] "Opened local port" port={Description:nodePort for istio-system/istio-ingressgateway:status-port IP: IPFamily:4 Port:30082 Protocol:TCP}
I1214 18:44:12.396253   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.219.60:80/TCP/10.252.24.168:8080"
...
I1214 18:44:12.438090   98073 graceful_termination.go:173] "Deleting real server" realServer="10.252.182.0:8080/TCP/10.29.3.175:8080"
I1214 18:44:12.438528   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.185.21:5000/TCP/10.29.0.244:5000"
I1214 18:44:12.438536   98073 graceful_termination.go:159] "Trying to delete real server" realServer="10.252.185.21:5000/TCP/10.29.0.244:5000"
I1214 18:44:12.438565   98073 graceful_termination.go:170] "Skip deleting real server till all connection have expired" realServer="10.252.185.21:5000/TCP/10.29.0.244:5000" activeConnection=0 inactiveConnection=2
I1214 18:44:12.438586   98073 graceful_termination.go:153] "Adding real server to graceful delete real server list" realServer="10.252.185.21:5000/TCP/10.29.0.244:5000"
I1214 18:44:12.438593   98073 graceful_termination.go:65] "Adding real server to graceful delete real server list" realServer="10.252.185.21:5000/TCP/10.29.0.244:5000"
I1214 18:44:12.438777   98073 proxier.go:2135] "Using graceful delete" uniqueRealServer="10.252.252.230:80/TCP/10.252.25.251:80"
...
I1214 18:44:12.498256   98073 graceful_termination.go:159] "Trying to delete real server" realServer="10.252.144.108:3302/TCP/10.252.5.151:3302"
I1214 18:44:12.498280   98073 graceful_termination.go:173] "Deleting real server" realServer="10.252.144.108:3302/TCP/10.252.5.151:3302"

I1214 18:44:12.585072   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.128.10,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1214 18:44:12.588827   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.128.122,tcp:8080" ipSet="KUBE-CLUSTER-IP"
...
I1214 18:44:25.821744   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.4.9,tcp:8080,10.29.4.9" ipSet="KUBE-LOOP-BACK"
I1214 18:44:25.826332   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.5.17,tcp:7070,10.29.5.17" ipSet="KUBE-LOOP-BACK"
I1214 18:44:25.830215   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.5.3,tcp:8080,10.29.5.3" ipSet="KUBE-LOOP-BACK"
I1214 18:44:25.832294   98073 proxier.go:1607] "Restoring iptables" rules="*nat\n:KUBE-SERVICES - [0:0]\n:KUBE-POSTROUTING - [0:0]\n:KUBE-FIREWALL - [0:0]\n:KUBE-NODE-PORT - [0:0]\n:KUBE-LOAD-BALANCER - [0:0]\n:KUBE-MARK-MASQ - [0:0]\n-A KUBE-POSTROUTING -m comment --comment \"Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose\" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service lb portal\" -m set --match-set KUBE-LOAD-BALANCER dst,dst -j KUBE-LOAD-BALANCER\n-A KUBE-NODE-PORT -p tcp -m comment --comment \"Kubernetes nodeport TCP port for masquerade purpose\" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service cluster ip + port for masquerade purpose\" -m set --match-set KUBE-CLUSTER-IP src,dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT\n-A KUBE-LOAD-BALANCER -j KUBE-MARK-MASQ\n-A KUBE-FIREWALL -j KUBE-MARK-DROP\n-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT\n-A KUBE-SERVICES -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT\n-A KUBE-POSTROUTING -m mark ! --mark 0x00004000/0x00004000 -j RETURN\n-A KUBE-POSTROUTING -j MARK --xor-mark 0x00004000\n-A KUBE-POSTROUTING -m comment --comment \"kubernetes service traffic requiring SNAT\" -j MASQUERADE\n-A KUBE-MARK-MASQ -j MARK --or-mark 0x00004000\nCOMMIT\n*filter\n:KUBE-FORWARD - [0:0]\n:KUBE-NODE-PORT - [0:0]\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding rules\" -m mark --mark 0x00004000/0x00004000 -j ACCEPT\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding conntrack rule\" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT\n-A KUBE-NODE-PORT -m comment --comment \"Kubernetes health check node port\" -m set --match-set KUBE-HEALTH-CHECK-NODE-PORT dst -j ACCEPT\nCOMMIT\n"
I1214 18:44:25.832311   98073 iptables.go:422] running iptables-restore [-w --noflush --counters]
I1214 18:44:25.861848   98073 proxier.go:2157] "Delete service" virtualServer="10.252.252.127:8080/TCP"
I1214 18:44:25.861905   98073 proxier.go:2163] "Unbinding address" address="10.252.252.127"
I1214 18:44:25.862300   98073 proxier.go:2157] "Delete service" virtualServer="10.252.253.35:8080/TCP"
I1214 18:44:25.862316   98073 proxier.go:2163] "Unbinding address" address="10.252.253.35"
...
I1214 18:44:27.402947   98073 proxier.go:2157] "Delete service" virtualServer="10.252.140.186:80/TCP"
I1214 18:44:27.402965   98073 proxier.go:2163] "Unbinding address" address="10.252.140.186"
I1214 18:44:27.403078   98073 conntrack.go:66] Clearing conntrack entries [-D --orig-dst 10.252.250.96 -p udp]

I1214 18:44:27.518994   98073 pathrecorder.go:240] kube-proxy: "/metrics" satisfied by exact match
I1214 18:44:27.518994   98073 pathrecorder.go:240] kube-proxy: "/metrics" satisfied by exact match
I1214 18:44:27.546600   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.442771651s"
```
通过对kube-proxy日志的分析，发现这次是kube-proxy启动后的第一次规则同步，主要的时间花在了：
```console
I1214 18:44:12.585072   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.128.10,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1214 18:44:12.588827   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.128.122,tcp:8080" ipSet="KUBE-CLUSTER-IP"
...
I1214 18:44:25.821744   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.4.9,tcp:8080,10.29.4.9" ipSet="KUBE-LOOP-BACK"
I1214 18:44:25.826332   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.5.17,tcp:7070,10.29.5.17" ipSet="KUBE-LOOP-BACK"
```

需要注意的是这是因为重启kube proxy后的启动日志，因为是重启，所以节点上还存在旧的ip set规则，所以这里是在做ipset的清理工作，大约花费了13.3s左右。同步ipset的相关源码如下：

![kubeproxy-code1](/static/images/2022/kube-proxy-code1.png)

![kubeproxy-code2](/static/images/2022/kube-proxy-code2.png)

同时通过分析kube proxy日志也可以发现，kube proxy 删除了2901条KUBE-CLUSTER-IP ipset，
```console
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.1214.log | grep "Successfully deleted legacy ip set entry from ip set" | grep "KUBE-CLUSTER-IP" | wc -l
2901
```
每清理一条ipset需要花费3~5ms，所以会很耗时。

再看`I1214 18:44:42.571976   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.025248788s"`的这次延迟，相关的日志如下：
```console
I1214 18:44:27.597071   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:27.597083   98073 iptables.go:357] running iptables-save [-t filter]
I1214 18:44:27.598958   98073 iptables.go:357] running iptables-save [-t nat]
I1214 18:44:27.600469   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-MARK-DROP -t nat]
I1214 18:44:27.601449   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-SERVICES -t nat]
I1214 18:44:27.602487   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-POSTROUTING -t nat]
I1214 18:44:27.603273   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-FIREWALL -t nat]
I1214 18:44:27.604267   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-NODE-PORT -t nat]
I1214 18:44:27.605156   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-LOAD-BALANCER -t nat]
I1214 18:44:27.606006   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-MARK-MASQ -t nat]
I1214 18:44:27.606778   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-FORWARD -t filter]
I1214 18:44:27.607499   98073 iptables.go:462] running iptables: iptables [-w -N KUBE-NODE-PORT -t filter]
I1214 18:44:27.608242   98073 iptables.go:462] running iptables: iptables [-w -C OUTPUT -t nat -m comment --comment kubernetes service portals -j KUBE-SERVICES]
I1214 18:44:27.608964   98073 iptables.go:462] running iptables: iptables [-w -C PREROUTING -t nat -m comment --comment kubernetes service portals -j KUBE-SERVICES]
I1214 18:44:27.609937   98073 iptables.go:462] running iptables: iptables [-w -C POSTROUTING -t nat -m comment --comment kubernetes postrouting rules -j KUBE-POSTROUTING]
I1214 18:44:27.611046   98073 iptables.go:462] running iptables: iptables [-w -C FORWARD -t filter -m comment --comment kubernetes forwarding rules -j KUBE-FORWARD]
I1214 18:44:27.612232   98073 iptables.go:462] running iptables: iptables [-w -C INPUT -t filter -m comment --comment kubernetes health check rules -j KUBE-NODE-PORT]
I1214 18:44:27.751035   98073 proxier.go:1972] "Adding new service" serviceName="saas-jcpt-tomcat-qa/saas-jcpt-spf-platform-service-tomcat-qa:port-0" virtualServer="10.252.254.74:8080/TCP"
I1214 18:44:27.751065   98073 proxier.go:1996] "Bind address" address="10.252.254.74"
I1214 18:44:27.751429   98073 proxier.go:1972] "Adding new service" serviceName="saas-mcloud-tomcat-qa/saas-mcloud-mcloud-advertising-open-web-tomcat-qa:port-0" virtualServer="10.252.168.55:8080/TCP"
I1214 18:44:27.751444   98073 proxier.go:1996] "Bind address" address="10.252.168.55"
...
I1214 18:44:28.702163   98073 proxier.go:1972] "Adding new service" serviceName="ep-java-qa/multienvtest11-java-qa-standard" virtualServer="10.252.255.156:8080/TCP"
I1214 18:44:28.702178   98073 proxier.go:1996] "Bind address" address="10.252.255.156"
I1214 18:44:28.718324   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.193.209,tcp:8080,10.11.193.209" ipSet="KUBE-LOOP-BACK"
I1214 18:44:28.732345   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.193.7,tcp:10080,10.11.193.7" ipSet="KUBE-LOOP-BACK"
I1214 18:44:28.737782   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.193.92,tcp:8080,10.11.193.92" ipSet="KUBE-LOOP-BACK"
I1214 18:44:28.741814   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.194.129,tcp:8080,10.11.194.129" ipSet="KUBE-LOOP-BACK"
...
I1214 18:44:42.506040   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.252.255.98,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1214 18:44:42.511248   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="22780" ipSet="KUBE-NODE-PORT-TCP"
I1214 18:44:42.515278   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="32080" ipSet="KUBE-NODE-PORT-TCP"
I1214 18:44:42.519103   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="32443" ipSet="KUBE-NODE-PORT-TCP"
I1214 18:44:42.522707   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="39080" ipSet="KUBE-NODE-PORT-TCP"
I1214 18:44:42.529565   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.34.191,tcp:7071" ipSet="KUBE-LOAD-BALANCER"
I1214 18:44:42.533043   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.35.192,tcp:443" ipSet="KUBE-LOAD-BALANCER"
I1214 18:44:42.536317   98073 ipset.go:176] "Successfully added ip set entry to ip set" ipSetEntry="10.11.35.192,tcp:80" ipSet="KUBE-LOAD-BALANCER"
I1214 18:44:42.538789   98073 proxier.go:1607] "Restoring iptables" rules="*nat\n:KUBE-SERVICES - [0:0]\n:KUBE-POSTROUTING - [0:0]\n:KUBE-FIREWALL - [0:0]\n:KUBE-NODE-PORT - [0:0]\n:KUBE-LOAD-BALANCER - [0:0]\n:KUBE-MARK-MASQ - [0:0]\n-A KUBE-POSTROUTING -m comment --comment \"Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose\" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service lb portal\" -m set --match-set KUBE-LOAD-BALANCER dst,dst -j KUBE-LOAD-BALANCER\n-A KUBE-NODE-PORT -p tcp -m comment --comment \"Kubernetes nodeport TCP port for masquerade purpose\" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service cluster ip + port for masquerade purpose\" -m set --match-set KUBE-CLUSTER-IP src,dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT\n-A KUBE-LOAD-BALANCER -j KUBE-MARK-MASQ\n-A KUBE-FIREWALL -j KUBE-MARK-DROP\n-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT\n-A KUBE-SERVICES -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT\n-A KUBE-POSTROUTING -m mark ! --mark 0x00004000/0x00004000 -j RETURN\n-A KUBE-POSTROUTING -j MARK --xor-mark 0x00004000\n-A KUBE-POSTROUTING -m comment --comment \"kubernetes service traffic requiring SNAT\" -j MASQUERADE\n-A KUBE-MARK-MASQ -j MARK --or-mark 0x00004000\nCOMMIT\n*filter\n:KUBE-FORWARD - [0:0]\n:KUBE-NODE-PORT - [0:0]\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding rules\" -m mark --mark 0x00004000/0x00004000 -j ACCEPT\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding conntrack rule\" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT\n-A KUBE-NODE-PORT -m comment --comment \"Kubernetes health check node port\" -m set --match-set KUBE-HEALTH-CHECK-NODE-PORT dst -j ACCEPT\nCOMMIT\n"
I1214 18:44:42.538802   98073 iptables.go:422] running iptables-restore [-w --noflush --counters]
I1214 18:44:42.540390   98073 proxier.go:1620] "Network programming" endpoint="sn-java-qa/nbiz-fulfillcenter-java-qa-standard" elapsed=27.540364073
I1214 18:44:42.571976   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.025248788s"
```
通过对kube-proxy日志的分析，发现这次是kube-proxy启动后的第二次规则同步，主要的时间花在了"Successfully added ip set entry to ip set"，总共有2901条：
```console
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.1214.log | grep "Successfully added ip set entry to ip set" | grep "KUBE-CLUSTER-IP" | wc -l
2901
```

连起来一块看下，kube proxy在重启时的逻辑是：
- kube proxy启动后第一次规则同步时，删除了2901条ipset规则
- kube proxy启动后第二次规则同步时，增加了2901条ipset规则

那么为什么要删除2901条KUBE-CLUSTER-IP呢？继续看kube proxy源码：

- syncProxyRules在同步规则前，会通过`serviceUpdateResult := proxier.serviceMap.Update(proxier.serviceChanges)`更新service信息

```golang
// This is where all of the ipvs calls happen.
func (proxier *Proxier) syncProxyRules() {
	proxier.mu.Lock()
	defer proxier.mu.Unlock()

	// don't sync rules till we've received services and endpoints
	if !proxier.isInitialized() {
		klog.V(2).InfoS("Not syncing ipvs rules until Services and Endpoints have been received from master")
		return
	}

	// Keep track of how long syncs take.
	start := time.Now()
	defer func() {
		metrics.SyncProxyRulesLatency.Observe(metrics.SinceInSeconds(start))
		klog.V(4).InfoS("syncProxyRules complete", "elapsed", time.Since(start))
	}()

	// We assume that if this was called, we really want to sync them,
	// even if nothing changed in the meantime. In other words, callers are
	// responsible for detecting no-op changes and not calling this function.
	serviceUpdateResult := proxier.serviceMap.Update(proxier.serviceChanges)
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)

	staleServices := serviceUpdateResult.UDPStaleClusterIP
	// merge stale services gathered from updateEndpointsMap
	for _, svcPortName := range endpointUpdateResult.StaleServiceNames {
		...
	}

	klog.V(3).InfoS("Syncing ipvs proxier rules")
    ...
```
- `proxier.serviceMap.Update(proxier.serviceChanges)`调用的是service.go内的Update，Update关键的代码是`sm.apply(changes, result.UDPStaleClusterIP)`，将变动的service信息合并到serviceMap内
![kubeproxy-code3](/static/images/2022/kube-proxy3.png)

- sm.apply通过遍历变动的service，通过sm.merge及sm.umerge将更新的service信息同步到serviceMap内
![kubeproxy-code4](/static/images/2022/kube-proxy4.png)

- sm.merge将变动的service更新到serviceMap内，并打印了相关的日志信息
![kubeproxy-code5](/static/images/2022/kube-proxy5.png)

- sm.umerge将删除的service从serviceMap内删除
![kubeproxy-code6](/static/images/2022/kube-proxy6.png)

在kube proxy日志内过滤sm.merge内打印的`Adding new service port`信息，发现总共只有1636条，说明只有1636条service被增加到了serviceMap内：

```
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.1214.log | grep "I1214 18:44:12" | grep "Adding new service port" | wc -l
1636
```

1636加上上面kube proxy日志内删除的2901条KUBE-CLUSTER-IP ipset，正好与总的KUBE-CLUSTER-IP ipset数基本一致(有误差是因为有些是none类型的service，没有CLUSTER IP，不需要增加KUBE-CLUSTER-IP ipset)：
```console
[root@sh-saas-k8s1-node-qa-100 ~]# ipset list KUBE-CLUSTER-IP | grep "10." | wc -l
4533
```

为什么有些节点在kube proxy重启时延迟很低呢？查了一台启动时延迟低的节点日志：
```console
[root@sh-saas-k8s1-node-qa-111 kubernetes]# cat kube-proxy.1214.log | grep "I1214 18:34:04" | grep "Adding new service port" | wc -l
4537
```
可以看到，在这台延迟低的节点上，其增加了4537条service。

那么现在基本上可以确定kube proxy启动时第一次规则同步延迟低的原因了：

kube proxy启动时，通过event接收service的变动信息时有快有慢，在执行规则同步前如果所有的service信息都更新了，因为重启时ip set的规则在节点上都是存在的，所以就不需要再更新ipset，因此执行速度很快，延迟很低。否则就像上面看到的一样，延迟就会很高。


那么为什么kube proxy第二次规则同步时，kube proxy要去增加2901条KUBE-CLUSTER-IP呢？

原因很简单，因为kube proxy在第一次规则同步时，将2901条ipset规则删除了，到第二次规则同步时，因为serviceMap内已经更新到了全量的service条目，所以就会把删除的ipset规则全部增加上去。这也是kube proxy第二次规则同步延迟高的原因。

kube proxy启动之后，ipset的管理是动态维护的，下面是第二天的清理日志，可以看到条目很少：

```console
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.INFO | grep "ipset.go:168"  | grep "I1215"
I1215 00:00:02.306662   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.184.105,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1215 00:00:02.787688   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.172.243,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1215 00:00:02.791131   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.254.241,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1215 00:00:03.349655   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.240.177,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1215 00:00:05.888785   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.140.30,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1215 00:00:05.892685   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.198.50,tcp:8080" ipSet="KUBE-CLUSTER-IP"
I1215 05:00:38.763694   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.4.89,tcp:8080,10.29.4.89" ipSet="KUBE-LOOP-BACK"
I1215 10:11:47.418368   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.2.173,tcp:5000,10.29.2.173" ipSet="KUBE-LOOP-BACK"
I1215 10:20:28.844829   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.3.122,tcp:8080,10.29.3.122" ipSet="KUBE-LOOP-BACK"
I1215 10:33:02.984232   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.1.29,tcp:80,10.29.1.29" ipSet="KUBE-LOOP-BACK"
I1215 10:58:00.664531   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.7.35,tcp:8080,10.29.7.35" ipSet="KUBE-LOOP-BACK"
I1215 11:27:41.540962   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.0.67,tcp:80,10.29.0.67" ipSet="KUBE-LOOP-BACK"
I1215 13:51:43.297171   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.12.48,tcp:80,10.29.12.48" ipSet="KUBE-LOOP-BACK"
I1215 14:04:41.765484   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.4.242,tcp:8080,10.29.4.242" ipSet="KUBE-LOOP-BACK"
I1215 14:10:03.512859   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.1.126,tcp:8080,10.29.1.126" ipSet="KUBE-LOOP-BACK"
I1215 14:47:13.574596   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.12.48,tcp:8080,10.29.12.48" ipSet="KUBE-LOOP-BACK"
I1215 14:49:40.801388   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.3.52,tcp:8080,10.29.3.52" ipSet="KUBE-LOOP-BACK"
I1215 15:02:43.460845   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.252.144.228,tcp:8081" ipSet="KUBE-CLUSTER-IP"
I1215 15:04:08.729336   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.11.195.33,tcp:8082,10.11.195.33" ipSet="KUBE-LOOP-BACK"
I1215 15:05:11.886161   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.4.220,tcp:7070,10.29.4.220" ipSet="KUBE-LOOP-BACK"
I1215 15:08:44.950507   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.2.153,tcp:8012,10.29.2.153" ipSet="KUBE-LOOP-BACK"
I1215 15:08:44.954242   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.2.153,tcp:8022,10.29.2.153" ipSet="KUBE-LOOP-BACK"
I1215 15:08:44.957974   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.2.153,tcp:8112,10.29.2.153" ipSet="KUBE-LOOP-BACK"
I1215 15:08:44.961549   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.2.153,tcp:9090,10.29.2.153" ipSet="KUBE-LOOP-BACK"
I1215 15:08:44.965754   98073 ipset.go:168] "Successfully deleted legacy ip set entry from ip set" ipSetEntry="10.29.2.153,tcp:9091,10.29.2.153" ipSet="KUBE-LOOP-BACK"
```

另外发现一个奇怪的问题，顺便提一下，在查看kube proxy日志时发现syncProxyRules可以同时运行：
```console
[root@sh-saas-k8s1-node-qa-100 kubernetes]# cat kube-proxy.1214.log | grep -E "(Syncing|syncProxyRules|Loop|periodic|retrying)"
I1214 18:44:11.871526   98073 bounded_frequency_runner.go:192] sync-runner Loop running
I1214 18:44:11.871538   98073 bounded_frequency_runner.go:192] sync-runner Loop running
I1214 18:44:12.168986   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:27.546600   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.442771651s"
I1214 18:44:27.546625   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:27.597071   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:27.769946   98073 proxier.go:1008] "syncProxyRules complete" elapsed="223.309494ms"
I1214 18:44:41.872216   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:42.155196   98073 proxier.go:1008] "syncProxyRules complete" elapsed="282.981368ms"
I1214 18:44:42.155211   98073 bounded_frequency_runner.go:296] sync-runner: ran, next possible in 0s, periodic in 30s
I1214 18:44:42.571976   98073 proxier.go:1008] "syncProxyRules complete" elapsed="15.025248788s"
I1214 18:44:42.571988   98073 bounded_frequency_runner.go:296] sync-runner: ran, next possible in 0s, periodic in 30s
I1214 18:44:42.573505   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:43.052002   98073 proxier.go:1008] "syncProxyRules complete" elapsed="479.977306ms"
I1214 18:44:43.052018   98073 bounded_frequency_runner.go:296] sync-runner: ran, next possible in 0s, periodic in 30s
I1214 18:44:43.054763   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:43.604485   98073 proxier.go:1008] "syncProxyRules complete" elapsed="552.432929ms"
I1214 18:44:43.604500   98073 bounded_frequency_runner.go:296] sync-runner: ran, next possible in 0s, periodic in 30s
```
通过上面的日志可以看到（I1214 18:44:27.546625以及I1214 18:44:27.597071同在运行），同时启动了2个sync-runner Loop，大致看了下代码，新版本的k8s版本已经支持IP双栈，kube proxy默认就会按支持IP双栈运行，所以启动了2个proxier:
```console
I1214 18:44:27.546625   98073 proxier.go:1032] "Syncing ipvs proxier rules"
I1214 18:44:27.546630   98073 iptables.go:357] running ip6tables-save [-t filter]
I1214 18:44:27.546766   98073 service.go:419] "Adding new service port" portName="public-fe-node-qa/public-fe-zhan-h5-node-qa:http-7090" servicePort="10.252.254.75:7090/TCP"
I1214 18:44:27.546783   98073 service.go:419] "Adding new service port" portName="saas-caiwu-tomcat-qa/saas-caiwu-wmpay-adapter-tomcat-qa:port-0" servicePort="10.252.254.254:8080/TCP"
```
`I1214 18:44:27.546625   98073 proxier.go:1032] "Syncing ipvs proxier rules"`后面紧接着来了一条`iptables.go:357] running ip6tables-save [-t filter]`，说明这是IPV6 proxier的goroutine。

## 4.2 kube proxy运行时同步规则延迟高原因分析
先看10.11.96.100这台：
```console
I1219 19:00:12.625572  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.00417046s"
I1219 19:00:53.623217  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.058888462s"
I1219 19:01:17.093380  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.050323555s"
I1219 19:01:22.909836  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.352718929s"
I1219 19:01:27.091561  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.077237668s"
```
延迟高的时间范围在12.19 19:00~19:02。

看看延迟高的kube proxy日志：
```console
I1219 19:00:11.624913  455291 proxier.go:1032] "Syncing ipvs proxier rules"
I1219 19:00:11.624930  455291 iptables.go:357] running iptables-save [-t filter]
I1219 19:00:11.628025  455291 iptables.go:357] running iptables-save [-t nat]
I1219 19:00:11.638297  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-MARK-DROP -t nat]
I1219 19:00:11.638649  455291 config.go:262] "Calling handler.OnEndpointSliceUpdate"
I1219 19:00:11.644166  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-SERVICES -t nat]
I1219 19:00:11.649163  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-POSTROUTING -t nat]
I1219 19:00:11.650444  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-FIREWALL -t nat]
I1219 19:00:11.651926  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-NODE-PORT -t nat]
I1219 19:00:11.652621  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-LOAD-BALANCER -t nat]
I1219 19:00:11.653556  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-MARK-MASQ -t nat]
I1219 19:00:11.654549  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-FORWARD -t filter]
I1219 19:00:11.655722  455291 iptables.go:462] running iptables: iptables [-w -N KUBE-NODE-PORT -t filter]
I1219 19:00:11.658151  455291 iptables.go:462] running iptables: iptables [-w -C OUTPUT -t nat -m comment --comment kubernetes service portals -j KUBE-SERVICES]
I1219 19:00:11.661089  455291 iptables.go:462] running iptables: iptables [-w -C PREROUTING -t nat -m comment --comment kubernetes service portals -j KUBE-SERVICES]
I1219 19:00:11.662551  455291 iptables.go:462] running iptables: iptables [-w -C POSTROUTING -t nat -m comment --comment kubernetes postrouting rules -j KUBE-POSTROUTING]
I1219 19:00:11.663517  455291 iptables.go:462] running iptables: iptables [-w -C FORWARD -t filter -m comment --comment kubernetes forwarding rules -j KUBE-FORWARD]
I1219 19:00:11.666142  455291 iptables.go:462] running iptables: iptables [-w -C INPUT -t filter -m comment --comment kubernetes health check rules -j KUBE-NODE-PORT]
I1219 19:00:11.937397  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for istio-system/istio-ingressgateway:status-port IP: IPFamily:4 Port:30082 Protocol:TCP}
I1219 19:00:11.971840  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for kube-system/kubernetes-dashboard IP: IPFamily:4 Port:27651 Protocol:TCP}
I1219 19:00:12.059729  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for kube-system/traefik-ingress:websecure IP: IPFamily:4 Port:32443 Protocol:TCP}
I1219 19:00:12.133834  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for weimobcloud-lua-qa/apisix-server-lua-qa IP: IPFamily:4 Port:39080 Protocol:TCP}
I1219 19:00:12.185355  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for kube-system/traefik-ingress:web IP: IPFamily:4 Port:32080 Protocol:TCP}
I1219 19:00:12.333484  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for fe-node-qa/bos-fe-xapi-node-node-qa-standard IP: IPFamily:4 Port:22780 Protocol:TCP}
I1219 19:00:12.366691  455291 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for istio-system/istio-ingressgateway:http2 IP: IPFamily:4 Port:30080 Protocol:TCP}
I1219 19:00:12.565088  455291 proxier.go:1607] "Restoring iptables" rules="*nat\n:KUBE-SERVICES - [0:0]\n:KUBE-POSTROUTING - [0:0]\n:KUBE-FIREWALL - [0:0]\n:KUBE-NODE-PORT - [0:0]\n:KUBE-LOAD-BALANCER - [0:0]\n:KUBE-MARK-MASQ - [0:0]\n-A KUBE-POSTROUTING -m comment --comment \"Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose\" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service lb portal\" -m set --match-set KUBE-LOAD-BALANCER dst,dst -j KUBE-LOAD-BALANCER\n-A KUBE-NODE-PORT -p tcp -m comment --comment \"Kubernetes nodeport TCP port for masquerade purpose\" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service cluster ip + port for masquerade purpose\" -m set --match-set KUBE-CLUSTER-IP src,dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT\n-A KUBE-LOAD-BALANCER -j KUBE-MARK-MASQ\n-A KUBE-FIREWALL -j KUBE-MARK-DROP\n-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT\n-A KUBE-SERVICES -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT\n-A KUBE-POSTROUTING -m mark ! --mark 0x00004000/0x00004000 -j RETURN\n-A KUBE-POSTROUTING -j MARK --xor-mark 0x00004000\n-A KUBE-POSTROUTING -m comment --comment \"kubernetes service traffic requiring SNAT\" -j MASQUERADE\n-A KUBE-MARK-MASQ -j MARK --or-mark 0x00004000\nCOMMIT\n*filter\n:KUBE-FORWARD - [0:0]\n:KUBE-NODE-PORT - [0:0]\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding rules\" -m mark --mark 0x00004000/0x00004000 -j ACCEPT\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding conntrack rule\" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT\n-A KUBE-NODE-PORT -m comment --comment \"Kubernetes health check node port\" -m set --match-set KUBE-HEALTH-CHECK-NODE-PORT dst -j ACCEPT\nCOMMIT\n"
I1219 19:00:12.565107  455291 iptables.go:422] running iptables-restore [-w --noflush --counters]
I1219 19:00:12.573243  455291 proxier.go:1620] "Network programming" endpoint="public-tjpm-java-qa/public-tjpm-hanoi-server-emr-java-qa" elapsed=1.573189717
I1219 19:00:12.625572  455291 proxier.go:1008] "syncProxyRules complete" elapsed="1.00417046s"
```
发现主要是在"Port was open before and is still needed"时稍慢，看看节点当时的资源使用情况：
![kubeproxy-code7](/static/images/2022/kube-proxy7.png)
当时CPU资源基本用完了，初步判断可能是节点负载高导致的。

再看另一台节点10.11.96.98：
```console
I1214 19:27:44.586760  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.015047146s"
I1215 15:02:33.378674  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.306158272s"
I1215 17:51:58.651549  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.090296314s"
I1215 18:55:27.000296  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.169267291s"
I1215 18:55:28.319427  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.319091838s"
I1215 19:01:18.928222  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.147711822s"
I1216 14:41:18.297584  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.196060478s"
I1216 14:52:36.661525  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.110995506s"
I1216 14:52:37.987179  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.325606522s"
I1216 15:38:09.018547  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.405135864s"
I1216 16:05:06.535796  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.425388635s"
I1219 15:06:22.626687  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.131007056s"
```
这台延迟高的频率多一些，我们看下15至16号的资源使用情况：
![kubeproxy-code8](/static/images/2022/kube-proxy8.png)
看起来CPU资源虽然偏高，但并没有用尽，不像是节点负载高导致的。

再看看kube proxy的日志：
```console
I1215 15:02:32.075482  250794 proxier.go:1032] "Syncing ipvs proxier rules"
I1215 15:02:32.075495  250794 iptables.go:357] running iptables-save [-t filter]
I1215 15:02:32.078397  250794 iptables.go:357] running iptables-save [-t nat]
I1215 15:02:32.080682  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-MARK-DROP -t nat]
I1215 15:02:32.081961  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-SERVICES -t nat]
I1215 15:02:32.093277  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-POSTROUTING -t nat]
I1215 15:02:32.095125  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-FIREWALL -t nat]
I1215 15:02:32.097053  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-NODE-PORT -t nat]
I1215 15:02:32.100061  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-LOAD-BALANCER -t nat]
I1215 15:02:32.104024  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-MARK-MASQ -t nat]
I1215 15:02:32.110049  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-FORWARD -t filter]
I1215 15:02:32.113117  250794 iptables.go:462] running iptables: iptables [-w -N KUBE-NODE-PORT -t filter]
I1215 15:02:32.116829  250794 config.go:262] "Calling handler.OnEndpointSliceUpdate"
I1215 15:02:32.120106  250794 iptables.go:462] running iptables: iptables [-w -C OUTPUT -t nat -m comment --comment kubernetes service portals -j KUBE-SERVICES]
I1215 15:02:32.125820  250794 iptables.go:462] running iptables: iptables [-w -C PREROUTING -t nat -m comment --comment kubernetes service portals -j KUBE-SERVICES]
I1215 15:02:32.127091  250794 iptables.go:462] running iptables: iptables [-w -C POSTROUTING -t nat -m comment --comment kubernetes postrouting rules -j KUBE-POSTROUTING]
I1215 15:02:32.130808  250794 iptables.go:462] running iptables: iptables [-w -C FORWARD -t filter -m comment --comment kubernetes forwarding rules -j KUBE-FORWARD]
I1215 15:02:32.134974  250794 iptables.go:462] running iptables: iptables [-w -C INPUT -t filter -m comment --comment kubernetes health check rules -j KUBE-NODE-PORT]
I1215 15:02:32.421079  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for kube-system/traefik-ingress:web IP: IPFamily:4 Port:32080 Protocol:TCP}
I1215 15:02:32.569211  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for istio-system/istio-ingressgateway:status-port IP: IPFamily:4 Port:30082 Protocol:TCP}
I1215 15:02:32.580709  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for kube-system/traefik-ingress:websecure IP: IPFamily:4 Port:32443 Protocol:TCP}
I1215 15:02:32.612355  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for istio-system/istio-ingressgateway:http2 IP: IPFamily:4 Port:30080 Protocol:TCP}
I1215 15:02:32.782457  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for fe-node-qa/bos-fe-xapi-node-node-qa-standard IP: IPFamily:4 Port:22780 Protocol:TCP}
I1215 15:02:32.827544  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for kube-system/kubernetes-dashboard IP: IPFamily:4 Port:27651 Protocol:TCP}
I1215 15:02:33.091202  250794 proxier.go:1434] "Port was open before and is still needed" port={Description:nodePort for weimobcloud-lua-qa/apisix-server-lua-qa IP: IPFamily:4 Port:39080 Protocol:TCP}
I1215 15:02:33.153133  250794 config.go:262] "Calling handler.OnEndpointSliceUpdate"
I1215 15:02:33.174825  250794 config.go:262] "Calling handler.OnEndpointSliceUpdate"
I1215 15:02:33.278997  250794 proxier.go:1607] "Restoring iptables" rules="*nat\n:KUBE-SERVICES - [0:0]\n:KUBE-POSTROUTING - [0:0]\n:KUBE-FIREWALL - [0:0]\n:KUBE-NODE-PORT - [0:0]\n:KUBE-LOAD-BALANCER - [0:0]\n:KUBE-MARK-MASQ - [0:0]\n-A KUBE-POSTROUTING -m comment --comment \"Kubernetes endpoints dst ip:port, source ip for solving hairpin purpose\" -m set --match-set KUBE-LOOP-BACK dst,dst,src -j MASQUERADE\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service lb portal\" -m set --match-set KUBE-LOAD-BALANCER dst,dst -j KUBE-LOAD-BALANCER\n-A KUBE-NODE-PORT -p tcp -m comment --comment \"Kubernetes nodeport TCP port for masquerade purpose\" -m set --match-set KUBE-NODE-PORT-TCP dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m comment --comment \"Kubernetes service cluster ip + port for masquerade purpose\" -m set --match-set KUBE-CLUSTER-IP src,dst -j KUBE-MARK-MASQ\n-A KUBE-SERVICES -m addrtype --dst-type LOCAL -j KUBE-NODE-PORT\n-A KUBE-LOAD-BALANCER -j KUBE-MARK-MASQ\n-A KUBE-FIREWALL -j KUBE-MARK-DROP\n-A KUBE-SERVICES -m set --match-set KUBE-CLUSTER-IP dst,dst -j ACCEPT\n-A KUBE-SERVICES -m set --match-set KUBE-LOAD-BALANCER dst,dst -j ACCEPT\n-A KUBE-POSTROUTING -m mark ! --mark 0x00004000/0x00004000 -j RETURN\n-A KUBE-POSTROUTING -j MARK --xor-mark 0x00004000\n-A KUBE-POSTROUTING -m comment --comment \"kubernetes service traffic requiring SNAT\" -j MASQUERADE\n-A KUBE-MARK-MASQ -j MARK --or-mark 0x00004000\nCOMMIT\n*filter\n:KUBE-FORWARD - [0:0]\n:KUBE-NODE-PORT - [0:0]\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding rules\" -m mark --mark 0x00004000/0x00004000 -j ACCEPT\n-A KUBE-FORWARD -m comment --comment \"kubernetes forwarding conntrack rule\" -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT\n-A KUBE-NODE-PORT -m comment --comment \"Kubernetes health check node port\" -m set --match-set KUBE-HEALTH-CHECK-NODE-PORT dst -j ACCEPT\nCOMMIT\n"
I1215 15:02:33.279013  250794 iptables.go:422] running iptables-restore [-w --noflush --counters]
I1215 15:02:33.291696  250794 proxier.go:1620] "Network programming" endpoint="knative-apps/saas-fe-mars-gateway-sssx4-private" elapsed=3.291653824
I1215 15:02:33.378674  250794 proxier.go:1008] "syncProxyRules complete" elapsed="1.306158272s"
```
可以看到时间同样是花在了"Port was open before and is still needed"，平均每条"Port was open before and is still needed"日志花费了100多ms。

但是不能就认为是这条日志本身花费了时间，应该理解为一个service的规则同步时间变长了。因为"Port was open before and is still needed"日志是每个service同步时都会输出的。

kube proxy运行时的这些高延迟因为缺少相关日志，原因待查。可能跟节点的资源抢占有一些关系。

kube proxy的规则同步平均延迟为500ms左右，目前发现的高延迟要比平均高2～3倍，时间为2s内，总体还是可以接受的。

## 4.3 监控显示延迟高实际延迟并不高的情况分析

10.11.96.111这台节点监控显示在19:06延迟为16s：

![kubeproxy2](/static/images/2022/kube-proxy2.png)

监控图的PromSQL如下：
`histogram_quantile(0.99,rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket{job="kube-proxy", instance=~"$instance"}[5m]))`

查看kube proxy日志：
```console
[root@sh-saas-k8s1-node-qa-111 kubernetes]# cat kube-proxy.INFO | grep "1214 19:06" | grep syncProxyRules
I1214 19:06:00.986442  614210 proxier.go:1008] "syncProxyRules complete" elapsed="451.730182ms"
I1214 19:06:01.500871  614210 proxier.go:1008] "syncProxyRules complete" elapsed="514.37159ms"
I1214 19:06:11.057996  614210 proxier.go:1008] "syncProxyRules complete" elapsed="114.4089ms"
I1214 19:06:14.699796  614210 proxier.go:1008] "syncProxyRules complete" elapsed="437.211872ms"
I1214 19:06:16.449299  614210 proxier.go:1008] "syncProxyRules complete" elapsed="430.982704ms"
I1214 19:06:16.868858  614210 proxier.go:1008] "syncProxyRules complete" elapsed="419.485422ms"
I1214 19:06:26.945975  614210 proxier.go:1008] "syncProxyRules complete" elapsed="388.775549ms"
I1214 19:06:34.474405  614210 proxier.go:1008] "syncProxyRules complete" elapsed="454.833547ms"
I1214 19:06:34.903892  614210 proxier.go:1008] "syncProxyRules complete" elapsed="429.444449ms"
I1214 19:06:37.054051  614210 proxier.go:1008] "syncProxyRules complete" elapsed="428.993622ms"
I1214 19:06:37.479665  614210 proxier.go:1008] "syncProxyRules complete" elapsed="425.573165ms"
I1214 19:06:38.439900  614210 proxier.go:1008] "syncProxyRules complete" elapsed="457.753396ms"
I1214 19:06:41.211820  614210 proxier.go:1008] "syncProxyRules complete" elapsed="153.288153ms"
I1214 19:06:57.251040  614210 proxier.go:1008] "syncProxyRules complete" elapsed="458.694322ms"
```
可以看到19:06分的规则同步时间没有超过600ms的。


histogram类型的指标还有Summary可以查询准确的平均时间：
![kubeproxy12](/static/images/2022/kube-proxy12.png)
可以看到这段时间的平均延迟为340ms左右。
监控图的PromSQL如下：
`kubeproxy_sync_proxy_rules_duration_seconds_sum{job="kube-proxy", instance=~"$instance"} / kubeproxy_sync_proxy_rules_duration_seconds_count{job="kube-proxy", instance=~"$instance"}`


不管是从日志还是Summary的监控数据都说明，kubeproxy_sync_proxy_rules_duration_seconds_bucket这个指标的数据是有问题的。

仔细查询19:06左右的监控数据，下面是19:05:30的:
![kubeproxy11](/static/images/2022/kube-proxy11.png)
19:06:00:
![kubeproxy9](/static/images/2022/kube-proxy9.png)
19:06:30:
![kubeproxy10](/static/images/2022/kube-proxy10.png)
根据histogram类型的指标规则,更大的桶的数据是比其更小的桶的数据的累加，而19:06:00的数据le=8.12及其后面的指标数据比更小的桶的数据还更小。确实是kubeproxy_sync_proxy_rules_duration_seconds_bucket指标的数据有问题，属于监控数据错误。

至于为什么监控数据会有问题，这是另一个问题了，另找时间排查promethues监控系统。


# 5. 结论

- kube proxy在重启或启动时需要同步全量的service的Cluster IP到ipset内，而操作ipset是很慢的，平均每条执行时间为3~5ms
- kube proxy重启时，service event更新的快慢（同时需注意为异步）会导致规则同步的延迟速度，同时也说明当service量大时，重启kube proxy是有一定的断流风险，在4500左右的service情况下，最久的断流时间大约在40s以内
- kube proxy启动完成后，在运行时偶尔也有响应慢的情况，时间在2s内，算是可接受的范围
- 通过kubeproxy_sync_proxy_rules_duration_seconds_bucket桶类型的指标的监控数据偶尔有问题，原因待查


# 6. 参考

kube proxy源码分析参考：[https://zhuanlan.zhihu.com/p/110860205](https://zhuanlan.zhihu.com/p/110860205)

kube proxy的metrics：[https://blog.51cto.com/u_13579597/5909948](https://blog.51cto.com/u_13579597/5909948)

https://blog.csdn.net/wtan825/article/details/94616813

https://p8s.io/docs/promql/query/histograms/

kube proxy的metrics及相关解释。
```console
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
gc时间

# HELP go_goroutines Number of goroutines that currently exist.
# TYPE go_goroutines gauge
goroutine数量

# HELP go_threads Number of OS threads created.
# TYPE go_threads gauge
线程数量

# HELP kubeproxy_network_programming_duration_seconds [ALPHA] In Cluster Network Programming Latency in seconds
# TYPE kubeproxy_network_programming_duration_seconds histogram
service或者pod发生变化到kube-proxy规则同步完成时间指标含义较复杂，参照https://github.com/kubernetes/community/blob/master/sig-scalability/slos/network_programming_latency.md

# HELP kubeproxy_sync_proxy_rules_duration_seconds [ALPHA] SyncProxyRules latency in seconds
# TYPE kubeproxy_sync_proxy_rules_duration_seconds histogram
规则同步耗时

# HELP kubeproxy_sync_proxy_rules_endpoint_changes_pending [ALPHA] Pending proxy rules Endpoint changes
# TYPE kubeproxy_sync_proxy_rules_endpoint_changes_pending gauge
endpoint 发生变化后规则同步pending的次数

# HELP kubeproxy_sync_proxy_rules_endpoint_changes_total [ALPHA] Cumulative proxy rules Endpoint changes
# TYPE kubeproxy_sync_proxy_rules_endpoint_changes_total counter
endpoint 发生变化后规则同步的总次数

# HELP kubeproxy_sync_proxy_rules_iptables_restore_failures_total [ALPHA] Cumulative proxy iptables restore failures
# TYPE kubeproxy_sync_proxy_rules_iptables_restore_failures_total counter
本机上 iptables restore 失败的总次数

# HELP kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds [ALPHA] The last time a sync of proxy rules was queued
# TYPE kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds gauge
最近一次规则同步的请求时间戳，如果比下一个指标 kubeproxy_sync_proxy_rules_last_timestamp_seconds 大很多，那说明同步 hung 住了

# HELP kubeproxy_sync_proxy_rules_last_timestamp_seconds [ALPHA] The last time proxy rules were successfully synced
# TYPE kubeproxy_sync_proxy_rules_last_timestamp_seconds gauge
最近一次规则同步的完成时间戳

# HELP kubeproxy_sync_proxy_rules_service_changes_pending [ALPHA] Pending proxy rules Service changes
# TYPE kubeproxy_sync_proxy_rules_service_changes_pending gauge
service变化引起的规则同步pending数量

# HELP kubeproxy_sync_proxy_rules_service_changes_total [ALPHA] Cumulative proxy rules Service changes
# TYPE kubeproxy_sync_proxy_rules_service_changes_total counter
service变化引起的规则同步总数

# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
利用这个指标统计cpu使用率

# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
进程可以打开的最大fd数

# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
进程当前打开的fd数

# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
统计内存使用大小

# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
进程启动时间戳

# HELP rest_client_request_duration_seconds [ALPHA] Request latency in seconds. Broken down by verb and URL.
# TYPE rest_client_request_duration_seconds histogram
请求 apiserver 的耗时(按照url和verb统计)

# HELP rest_client_requests_total [ALPHA] Number of HTTP requests, partitioned by status code, method, and host.
# TYPE rest_client_requests_total counter
请求 apiserver 的总数(按照code method host统计)
```
