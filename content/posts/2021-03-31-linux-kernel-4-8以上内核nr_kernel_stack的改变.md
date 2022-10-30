---
title: linux kernel 4.8以上内核NR_KERNEL_STACK的改变
author: 阿辉
date: 2021-03-31T09:39:13+00:00
categories:
- linux系统
- kernel
tags:
- kubernetes
- kernel
- linux系统
keywords:
- kubernetes
- kernel
- linux系统
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

# 1. NR_KERNEL_STACK是干什么用的？

通过下面的命令，我们可以查看内核栈的数量：
```
cat /proc/vmstat  | grep stack
nr_kernel_stack 567
```
这个值在内核中的宏为：NR_KERNEL_STACK，表示当前内核中有多少个内核栈。

我们使用这个数字来监控K8S节点上使用PID的数量，以避免PID被耗尽。

# 2. 那么PID与内核栈有什么关系呢？

在linux系统中，每个进程、子进程和线程在运行时都会有一个PID。这些进程或线程在运行时，因为CPU需要进行任务切换，在任务切换时就需要上下文交换，在上下文交换时就需要把当前进程的上下文压到内核栈内去，以便下次再运行时取出继续执行。

所以可以确定：每个进程、子进程和线程都会有一个内核栈。内核栈的数量与PID的数量大致相当。

注：基于linux内核的线程，比如java的线程与linux的线程是一一对应的，nodejs只使用了linux的进程，线程模型是其自己实现的，golang最特别，使用了多进程，每个进程上有多线程（基于内核），线程上还是自己实现的协程或者说goroute(可以理解为自己实现的线程)

<!--more-->

# 3. linux kernel 4.8以上内核NR_KERNEL_STACK的改变

最近我们升级了K8S节点上docker的版本，发现最新的docker 19.03的版本在4.4.x的内核上有一些问题，于是就把内核版本升级到了5.4.x。

升级后，我收到了nr_kernel_stack过大的报警，这个报警的阀值是10万。之前没有特殊情况，从未达到过这个量。如下图所示：

![image](/wp-content/uploads/2021/03/nr_stack.png)

在看了好几台节点，发现都是这个结果后，我基本上可以确定这是由于升级内核导致的。

于是我们查阅内核相关的补丁，发现了下面这个补丁：
[mm: Track NR_KERNEL_STACK in KiB instead of number of stacks ](https://lore.kernel.org/patchwork/patch/692334/)

英文大意是说：
内存管理：NR_KERNEL_STACK由之前的追踪内核栈的数量替换内核栈的容量(KiB)

再看补丁修改过的源码：
![image](/wp-content/uploads/2021/03/nr_stack_patch.png)

从图中可以看到，主要是把之前的`mod_zone_page_state(zone, NR_KERNEL_STACK, account)`换成了`mod_zone_page_state(zone, NR_KERNEL_STACK_KB,THREAD_SIZE / 1024 * account)`。
也就是说: 内核栈的容量 = 栈大小 * 栈数量 / 1024(1k)

简单换算一下：`NR_KERNEL_STACK = NR_KERNEL_STACK_KB / THREAD_SIZE * 1024`
再理解一遍：内核栈的数量 = 内核栈的容量 / 栈大小 * 1024(1k)

那么内核栈的又怎么看呢？
方法是`ulimit -s`:
```
[root@sh-saas-k8s1-master-dev-01 ~]# ulimit -a | grep stack
stack size              (kbytes, -s) 8192
[root@sh-saas-k8s1-master-dev-01 ~]# ulimit -s
8192
```

所以现在，我只需要把新内核的nr_kernel_stack的值除以8(8192/1024=8)就是跟之前的内核版本一样的了。

最后，经过查看各个内核版本的代码，发现这个补丁是在4.8的版本开始引入的。