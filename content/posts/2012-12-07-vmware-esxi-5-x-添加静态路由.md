---
title: Vmware ESXI 5.x 添加静态路由
author: 阿辉
date: 2012-12-07T08:59:31+00:00
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
Configuring additional gateways for vmkernel ports on an ESXi host  
Purpose  
This article provides a command to configure routes to additional gateways for vmkernel ports on an ESXi host.  
Resolution  
Unlike ESX, ESXi does not have a service console. The management network is on a vmkernel port and therefore uses the default vmkernel gateway. 

You can only add one gateway for the vmkernal interface through vSphere Client. However, you can configure additional gateways from the command line.  
<!--more-->

  
To configure a second gateway for the management network:

Open a console to the ESX or ESXi host. For more information, see Unable to connect to an ESX host using Secure Shell (SSH) (1003807) or Using Tech Support Mode in ESXi 4.1 (1017910).  

Run this command:
```
esxcfg-route -a <target network> <netmask> <gateway>
```

For example, to add a route to 192.168.100.0 network through 192.168.0.1, run one of these command:  

```
esxcfg-route -a 192.168.100.0/24 192.168.0.1  
esxcfg-route -a 192.168.100.0 255.255.255.0 192.168.0.1
```
Note: The changes are not seen or copied when using host profiles.  
Additional Information

The command esxcfg-route -l lists the current routing table.