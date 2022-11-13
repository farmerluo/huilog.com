---
title: VMware Esxi 内虚拟机单网卡配双IP的问题
author: 阿辉
date: 2012-01-18T14:47:00+00:00
categories:
- Vmware ESXI
tags:
- Vmware ESXI
keywords:
- Vmware ESXI
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
因为我们托管的机房是双IP双线，今天在一个VMware ESXi的虚拟机内把一块网卡配成双IP。发现ping不通网关了。同样的方法在物理机上配置是没问题的。所以怀疑是VMWare ESXi配置的问题。

进入VMWare ESXi的配置-》网络。找到对应的虚拟交换机，点属性。

把下图中的vSwitch和VM PublicNetwork中的默认策略—》安全中的杂乱模式，MAC地址更改都改成接受。
<!--more-->

![/wp-content/uploads/baiduhi/025c36d3d539b600161a2306e950352ac75cb738.jpg](/wp-content/uploads/baiduhi/025c36d3d539b600161a2306e950352ac75cb738.jpg)

![/uploads/baiduhi/aeaa9b504fc2d562d316ff5ce71190ef77c66cf2.jpg](/uploads/baiduhi/aeaa9b504fc2d562d316ff5ce71190ef77c66cf2.jpg)

做了如上更改后，就可以ping通网关了。
