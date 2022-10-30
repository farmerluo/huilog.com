---
title: AWS VPC CNI路由分析
author: 阿辉
date: 2022-05-12T18:27:58+00:00
categories:
- kubernetes
- CNI
tags:
- kubernetes
- CNI
keywords:
- kubernetes
- CNI
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

{{< toc >}}

amazon-vpc-cni-k8s是AWS基于VPC CNI的k8s网络插件，有着高性能及高度灵活的优点。项目地址：
https://github.com/aws/amazon-vpc-cni-k8s

下面我们通过分析其源码，查看实际上相关的CNI网络是怎么实现的。

# 1. AWS VPC CNI Agent node部分

Node上的Agent程序与路由相关的，主要做2块工作，一个是配置主网卡（SetupHostNetwork），一个是配置辅助网卡（setupENINetwork）。

## 1.1 SetupHostNetwork

针对host网络和主网卡Primary ENI做一些配置，Primary ENI: 主机的默认网卡,默认一般为eth0。

1. 如果启用ipv4和支持node port，设置RP Filter：

```golang
// 启用ConfigureRpFilter，设置net.ipv4.conf.{primaryIntf}.rp_filter为2
primaryIntfRPFilter := "net/ipv4/conf/" + primaryIntf + "/rp_filter"
echo 2 > primaryIntfRPFilter
```

2. 设置Primary eni网卡的mtu：
```
ip link set dev <link> mtu MTU
```

3. 先删除main表里优先级为1024的rule
```golang
// If this is a restart, cleanup previous rule first
ip rule del fwmark 0x80/0x80 pref 1024 table main
```

<!--more-->

4. 如果启用node port支持，添加基于fwmark的路由策略:

```golang
// defaultConnmark is the default value for the connmark described above. Note: the mark space is a little crowded,
// - kube-proxy uses 0x0000c000
// - Calico uses 0xffff0000.
// defaultConnmark = 0x80
ip rule add fwmark 0x80/0x80 pref 1024 table main
```

5. 如果启用ipv4且启用pod eni，则：
```golang
// Add new rule with higher priority
// 添加table为255的优先级为20策略路由
ip rule add pref 20 table local

// 删除table为255的优先级为0的策略路由
ip rule del pref 0 table local
```

6. 创建snat的iptables规则，这里省略


## 1.2 setupENINetwork
    
setupENINetwork为非Primary ENI的弹性网卡配置网络，每块弹性网卡都有一个主IP+N个辅助IP

>  eniName: 主机的默认网卡,默认一般为eth0+x，X为网卡序号，如eth1

1. 设置eni网卡的mtu
```
ip link set dev <eniName> mtu MTU
```

2. 让eni网卡up
```
ip link set dev <eniName> up
```

3. 删除eni网卡上所有已经存在的ip地址
```
ip add del <eniIP> dev <eniName> (if necessary)
```

4. 设置eni网卡的ip
```golang
// eniIP： primary IP of that ENI
ip add add <eniIP> dev <eniName>
```

5. 在deviceNumber + 1路由表里，删除已经存在默认路由，添加默认路由指向eni子网的网关
```golang
// tableNumber := deviceNumber + 1
// gw: 网关为Eni IP子网的第二个IP地址
// Add a direct link route for the host is ENI IP only
ip route add <gw>/32 scope link  dev <eniName> table tableNumber

// Route all other traffic via the host is ENI IP
// tableNumber := deviceNumber + 1
// gw: 网关为Eni IP子网的第二个IP地址
ip route add 0.0.0.0/0 scope 0 via <gw> dev <eniName> table tableNumber

// 在main路由表里删除源地址为eniIP，目标地址为eniSubnetCIDR的路由
ip route del eniSubnetCIDR src eniIP scope link table main 
```
        
        
# 2. AWS CNI二进制文件

CNI二进制文件主要是实现CNI标准规则的几个接口：

## 2.1 cmdAdd

1. grpc请求"/rpc.CNIBackend/AddNetwork"调用ipam分配ip

2. 构建hostVethName
```golang
// build hostVethName
// Note: the maximum length for linux interface name is 15
// 返回veth网卡名字，{conf.VethPrefix}+{hash({namespace}.{pod name})前10位}
// VethPrefix is the prefix to use when constructing the host-side
// veth device name. It should be no more than four characters, and
// defaults to 'eni'.
// 如: eni1234567890
```

3. 创建veth和容器内网卡，绑定容器网卡ip地址，容器内设置路由和其他内核参数
```golang
// Clean up if hostVeth exists.
// 如果hostVethName网卡存在，删除宿主机网卡hostVethName
ip link del <oldlink>
```

4.以下步骤在POD network namespce内执行

    1) 创建网卡
    
    // createVethContext.contVethName:调用CNI二进制文件时传过来的网卡名：一般为eth0
    ip link add {createVethContext.contVethName} type veth peer name {createVethContext.hostVethName} /* on host namespace */

    2) 启动网卡
    
    ip link set dev {createVethContext.hostVethName} up
    ip link set dev {createVethContext.contVethName} up

    3) 添加路由
    
    // Add a connected route to a dummy next hop (169.254.1.1 or fe80::1)
    // # ip route show
    // default via 169.254.1.1 dev eth0
    // 169.254.1.1 dev eth0
    // scope link = scope 253
    ip route add 169.254.1.1 scope link dev {createVethContext.contVethName}

    // Add a default route via dummy next hop(169.254.1.1 or fe80::1). Then all outgoing traffic will be routed by this
    // default route via dummy next hop (169.254.1.1 or fe80::1)
    // 添加默认路由
    // scope 0 = scope universe
    ip route add 0.0.0.0/0 scope 0 via 169.254.1.1

    4) 给容器里的网卡绑定ip
    
    // $addr: 调用ipam分配ip
    ip addr add $addr dev <createVethContext.contVethName>

    5) 添加arp表
    
    // $addr: gateway ip 169.254.1.1
    // $mac: hostVeth.Attrs().HardwareAddr
    ip neigh add $addr lladdr $mac nud permanent dev <createVethContext.contVethName>

    6) 将host端网卡移动到host命名空间
    
    // veth into the host namespace.
    // * move {createVethContext.hostVethName} to Pod's namespace hostNS */
    ip link set {createVethContext.hostVethName} netns {hostNS} 

5. 启动host网卡

```golang
// Explicitly set the veth to UP state, because netlink doesn't always do that on all the platforms with net.FlagUp.
// veth won't get a link local address unless it's set to UP state
ip link set dev {createVethContext.hostVethName} up
```

6. 添加用于host上访问pod的路由
```golang
// Add or replace route
// $addr: 调用ipam分配ip
ip route add $addr/32 dev {hostVethName} scope link /* add host route */
```

7. 添加用于访问pod流量的路由策略

```golang
// $addr: 调用ipam分配ip
// 512:	from all to 10.200.202.222 lookup main
ip rule del to $addr/32 pref 512 table main
ip rule add to $addr/32 pref 512 table main

// add from-pod rule, only need it when it is not primary ENI
// <tableNumber>里的路由应该是在ipam中添加的，不是cni插件
if deviceNumber > 0 {
	// add rule: 1536: from <podIP> use table <table>
	// $addr: 调用ipam分配ip
	// tableNumber := deviceNumber + 1
	// deviceNumber: 调用ipam分配ip时取到的
	// 1536: from 10.200.202.222 to all lookup <tableNumber>
	ip rule del from $addr/32 pref 1536 table tableNumber
	ip rule add from $addr/32 pref 1536 table tableNumber
}
```


# 3. 参考文件

* AWS 弹性网卡参考

https://aws.amazon.com/cn/premiumsupport/knowledge-center/ec2-ubuntu-secondary-network-interface/

* cni文档

https://github.com/containernetworking/cni/blob/spec-v1.0.0/SPEC.md
https://www.cni.dev/docs/

* libcni库

https://github.com/containernetworking/cni/tree/main/scripts
https://www.cni.dev/docs/spec-upgrades/#specific-guidance-for-plugins-written-in-go

* cni调用原理

http://www.noobyard.com/article/p-mjmyxamv-ob.html
https://blog.csdn.net/shida_csdn/article/details/79752411
https://segmentfault.com/a/1190000019956620
https://morningspace.github.io/tech/k8s-net-cni/