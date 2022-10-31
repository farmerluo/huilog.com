---
title: centos linux 7 更新第三方内核
author: 阿辉
date: 2019-03-15T10:26:58+00:00
categories:
- Linux
tags:
- Linux
keywords:
- Linux
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

centos linux 7本身的内核为3.10，比较老，很多新的特性都没有，可以考虑使用第三方的比较新的内核。如ELRepo仓库就提供了kernel 4.4.x的长期支持版本及5.0.x的最新版.

启用 ELRepo 仓库:
导入密钥：
`rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org`
安装yum源包：
`yum install -y https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm`

安装内核：
`yum --enablerepo=elrepo-kernel install kernel-lt -y`

配置启动内核,使用最新的就可以了：
`egrep ^menuentry /etc/grub2.cfg | cut -f 2 -d \'`
`grub2-set-default 0 `

kernel 4.4 对kdump参数有调整，需做如下修改：
`vim /etc/default/grub `
把
`crashkernel=auto`
改成：
`crashkernel=128M`

<!--more-->

可使用以下命令一键修改：
`sed -i 's/crashkernel=auto/crashkernel=128M/' /etc/default/grub`
否则使用新内核重启后kdump会报错。

生成新的配置文件
`grub2-mkconfig -o /boot/grub2/grub.cfg`

重启机器：
`reboot`

重启后可以看看kdump服务是否正常：
`systemctl status kdump`

以下命令可以测试做一次kernel dump:
```bash
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger
```

如果需要更新header包及devel包，操作可参考：
```bash
yum --enablerepo=elrepo-kernel install kernel-ml
yum --enablerepo=elrepo-kernel -y swap kernel-headers -- kernel-ml-headers
yum --enablerepo=elrepo-kernel -y swap kernel-tools-libs -- kernel-ml-tools-libs
yum --enablerepo=elrepo-kernel -y install kernel-ml-tools
yum --enablerepo=elrepo-kernel -y swap kernel-devel -- kernel-ml-devel
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
```
这样当系统出问题时，就能方便调试问题了。