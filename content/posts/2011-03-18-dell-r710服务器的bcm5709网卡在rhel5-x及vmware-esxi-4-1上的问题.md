---
title: dell R710服务器的BCM5709网卡在RHEL5.x及vmware esxi 4.1上的问题
author: 阿辉
date: 2011-03-18T17:50:00+00:00
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
机房用的都是Dell R710服务器，自带BCM5709网卡。
默认RHEL5.x系统自带的驱动对BCM5709的网卡支持不好，网卡一遇到流量比较大就会hung up。
这个问题涉及到 ACPI 电源管理，当网卡在正常工作的时候，会被 ACPI 误以为他闲着，从而把它给hung up。
在RHEL 5.x或Centos 5.x上，解决办法有两种：

1.用内核自带的驱动，修改内核参数，关闭acpi：
```bash
vi /boot/grub/grub.conf
title CentOS (2.6.18-164.6.1.el5)
root (hd0,0)
kernel /vmlinuz-2.6.18-164.6.1.el5 ro root=/dev/VolGroup00/LogVol00 rhgb quiet acpi=off noapic
initrd /initrd-2.6.18-164.6.1.el5.img
```
参数acpi=off noapic是要加上的，这个解决办法经测试网卡的流量最高跑到300多M没事。

<!--more-->

2.从Dell网站下载最新的驱动升级，过程就不写了。

这个问题在linux系统上好解决，但是今天却收到几封机房发的报警邮件，说我装着vmware esxi 4.1的Dell R710服务器丢包严重，高的时候有50%，我连上去看了一下，确实有丢包。查了下esxi的日志，也没看到什么错误，这样就怀疑也是BCM5709的网卡驱动的问题了。

然后就去www.vmware.com查，发现最新的是vmware esxi 4.1 update 1了，仔细查看Release Notes（https://www.vmware.com/support/vsphere4/doc/vsp_esxi41_u1_rel_notes.html），里面找到了BCM5709网卡的驱动更新：

ESXi hosts might not boot or cause some devices not to be accessible when using more than 6 bnx2 ports
An error message similar to the following is displayed in /var/log/messages of ESXi: CPU10:4118 - intr vector: 290:out of interrupt vectors. Before applying this fix, bnx2 devices in MSI-X mode and jumbo frame configuration support only 6 ports. The issue is resolved in this release. In this release, the bnx2 driver allocates only 1 RX queue in MSI-X mode, supports 16 ports, and saves memory resources.                  

看到网卡驱动有更新就准备先升级到vmware 4.1 update 1,找了台机器安装vmware-vsphere-cli,开始升级：

1.到vmware下载升级包：update-from-esxi4.1-4.1_update01.zip

2.关闭运行的所有虚拟机

3.把esxi 4.1设置为维护模式

4.在刚安装vmware-vsphere-cli的机器上用vmware-vsphere-cli升级：
```
vihostupdate --server x.x.x.x --username root --password  passwd -i -b update-from-esxi4.1-4.1_update01.zip  -B ESXi410-Update01
```
运行后显示：
Please wait patch installation is in progress ...
The update completed successfully, but the system needs to be rebooted for the changes to be effecti
ve.

5.reboot esxi

 

重启后ssh连到esxi的控制台:
```
~ # ethtool -i vmnic0
driver: bnx2
version: 2.0.7d-3vmw
firmware-version: 5.0.13 bc 5.0.11 NCSI 2.0.5
bus-info: 0000:01:00.0
```
而在升级之前看到的是：
```
~ # ethtool -i vmnic0
driver: bnx2
version: 2.0.7d-2vmw
firmware-version: 5.0.13 bc 5.0.11 NCSI 2.0.5
bus-info: 0000:01:00.0
```
不给力呀，主版本号居然是一样的，担心还是有问题，继续找驱动，发现vmware还真有：http://downloads.vmware.com/d/info/datacenter_downloads/vmware_vsphere_hypervisor_esxi/4_0#drivers_tools

VMware ESX/ESXi 4.1 Driver CD for Broadcom NetXtreme II Ethernet Network Controllers

VMware对 Broadcom NetXtreme II Ethernet Network Controllers专门做了一个ISO，看来这个网卡的驱动问题多多。在下面这个地址下载这个ISO：

http://downloads.vmware.com/d/details/esx41_broadcom_netextremeii_dt/ZCV0YnRlaHRidHdw

下载后解压出来，找到BCM-bnx2-2.0.15g.8.v41.1-offline_bundle-325733.zip，升级：
```
vihostupdate --server x.x.x.x --username root --password  passwd -i -b BCM-bnx2-2.0.15g.8.v41.1-offline_bundle-325733.zip
```
安装完后reboot esxi,

再看驱动版本：
```
~ # ethtool -i vmnic0
driver: bnx2
version: 2.0.15g.8.v41.1
firmware-version: 5.0.13 bc 5.0.11 NCSI 2.0.5
bus-info: 0000:01:00.0
```
这下感觉好多了，哈哈。

然后退出维护模式，启动虚拟机。

不知道效果怎么样，能不能解决问题，只能再观察了。