---
title: VMware ESX Server 命令行
author: 阿辉
date: 2009-07-17T11:37:00+00:00
categories:
- VMware
tags:
- VMware
keywords:
- VMware
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
VMware ESX Server 上特有的命令很多，以下分享一下常见的命令行的使用方法。
1：看你的esx版本。

`vmware -v`

2：列出esx里知道的服务

`esxcfg-firewall -s`

3：查看具体服务的情况

`esxcfg-firewall -q sshclinet`

4：重新启动vmware服务

`service mgmt-vmware restart`

5: 修改root的密码

`passwd root`

6：列出你当前的虚拟交换机

`esxcfg-vswitch -l`

<!--more-->
7：查看控制台的设置

`esxcfg-vswif -l`

8：列出系统的网卡

`esxcfg-nics -l`

9：添加一个虚拟交换机，名字叫（internal）连接到两块物理网卡，（重新启动服务，vi就能看见了）
```
esxcfg-vswitch -a vSwitch1
esxcfg-vswitch -A internal vSwitch1
esxcfg-vswitch -L vmnic1 vSwitch1
esxcfg-vswitch -L vmnic2 vSwitch1
```

10：删除交换机,(注意，别把控制台的交换机也删了）

`esxcfg-vswitch -D vSwitch1`

11：删除交换机上的网卡

`esxcfg-vswitch -u vmnic1 vswitch2`

12：删除portgroup

`esxcfg-vswitch -D internel vswitch1`

13：创建 vmkernel switch ，如果你希望使用vmotion，iscsi的这些功能，你必须创建( 通常是不需要添加网关的）
```
esxcfg-vswitch -l
esxcfg-vswitch -a vswitch2
esxcfg-vswitch -A "vm kernel" vswitch2
esxcfg-vswitch -L vmnic3 vswitch2
esxcfg-vmknic -a "vm kernel" -i 172.16.1.141 -n 255.255.252.0
esxcfg-route 172.16.0.254
```
14：打开防火墙ssh端口
```
esxcfg-firewall -e sshclient
esxcfg-firewall -d sshclient
```
15: 创建控制台
```
esxcfg-vswitch -a vSwitch0
esxcfg-vswitch -A "service console" vSwitch0
esxcfg-vswitch -L vmnic0 vSwitch0
esxcfg-vswif -a vswif0 -p "service console" -i 172.16.1.140 -n 255.255.252.0
```
16: 添加nas设备(a 添加标签，-o，是nas服务器的名字或ip，-s 是nas输入的共享名字）
```
esxcfg-nas -a isos -o nas.vmwar.cn -s isos
```
17：列出nas连接

`esxcfg-nas -l`

18: 强迫esx去连接nas服务器(用`esxcfg-nas -l` 来看看结果）
```
esxcfg-nas -r
esxcfg-nas -l
```
19：连接iscsi 设备(`e:enable q:`查询 `d：disable s:`强迫搜索）

`esxcfg-swiscsi -e`

20：设置targetip

`vmkiscsi-tool -D -a 172.16.1.133 vmhba40`

21：列出和target的连接

`vmkiscsi-tool -l -T vmhba40`

22：列出当前的磁盘

`ls -l /vmfs/devices/disks`

***************************管理安装在ESX上安装的虚拟机*******************************

在ESX中，主要是通过vmware-cmd这个命令来管理虚拟机的，包括虚拟机的开关、状态查询和添加删除虚拟设备。

1，列出所有虚拟机（这里列出的是所有虚拟机各自对应的配置文件，ESX技术通过修改这些配置文件来完成对虚拟机的管理的）：
```
    # vmware-cmd -l
    /vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx
```
2，查询某个虚拟机状态（是否加电）：
```
    # vmware-cmd /vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx getstate
    getstate() = on
```
3，启动/停止/重启/暂停某台虚拟机：
```
    # vmware-cmd /vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx start/stop/reset/suspend trysoft  
```
这里，trysoft是电源管理模式，意思是先尝试安全操作，如果失败则进行强制操作，除了这个选项，还用soft和hard这两种模式。
    start/stop/reset/suspend只能选择其一.
    
4，查询设备属性：
```
    # vmware-cmd //vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx getconfig ide0:0.deviceType
    getconfig(ide0:0.deviceType) = cdrom-raw
```
    这里，我们获得的是ide0:0设备的类型，由输出可知，该设备是CDROM，而且是虚拟一个物理光驱。

5，将4中的光驱改为可以读取ISO文件的虚拟光驱：
```
    # vmware-cmd //vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx setconfig ide0:0.deviceType cdrom-image
```
6，为虚拟光纤指定ISO文件
```
    # vmware-cmd //vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx setconfig ide0:0.fileName /export/images/Linux4-U3.iso
```
7，断开设备连接
```
    # vmware-cmd //vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx disconnectdevice ide0:0
```
8，连接设备：
```
    # vmware-cmd //vmfs/volumes/4655dd66-758d208c-1b24-001aa0187722/sol_vm1/sol_vm1.vmx connectdevice ide0:0
```
    这里列出的管理命令仅仅是ESX众多设备管理中的一部分，然而通过实践，用户可以利用命令行完成所以需要通过VM client可以实现的工作，而且用户可以更加理解ESX的实现原理，更好的维护自己的ESX操作系统。