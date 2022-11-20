---
title: 服务器报警，出现 blocked for more than 120 seconds.
author: 阿辉
date: 2010-07-20T15:44:00+00:00
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
前几天服务器发生了一次报警，显示系统负载过高,Nagios监控发生超时。等我连上去看的时候，已经收到服务器恢复正常，报警解除的短信了。

看了下日志,有以下信息：
<!--more-->
```
Jul 16 10:14:47 web3 nrpe[18939]: Error: Could not complete SSL handshake. 5
Jul 16 12:12:07 web3 rpc.statd[3007]: Received SM_UNMON request from web3.shinezone.com for 192.168.1.2 while not monitoring any hosts.
Jul 16 12:14:11 web3 kernel: INFO: task nfsd:3269 blocked for more than 120 seconds.
Jul 16 12:14:11 web3 kernel: “echo 0 > /proc/sys/kernel/hung_task_timeout_secs” disables this message.
Jul 16 12:14:11 web3 kernel: nfsd          D ffff81023dc3fb88     0  3269      1          3270  3268 (L-TLB)
Jul 16 12:14:11 web3 kernel:  ffff8104ac50dc40 0000000000000046 ffff8104ac50dc50 ffffffff80062ff8
Jul 16 12:14:11 web3 kernel:  0000000000000000 000000000000000a ffff8104ac91d820 ffff81032cf41820
Jul 16 12:14:11 web3 kernel:  0011e5ad2e3f154c 00000000000003fe ffff8104ac91da08 0000000200002f1d
Jul 16 12:14:11 web3 kernel: Call Trace:
Jul 16 12:14:11 web3 kernel:  [] thread_return+0x62/0xfe
Jul 16 12:14:11 web3 kernel:  [] :jbd:log_wait_commit+0xa3/0xf5
Jul 16 12:14:11 web3 kernel:  [] autoremove_wake_function+0x0/0x2e
Jul 16 12:14:11 web3 kernel:  [] process_timeout+0x0/0x5
Jul 16 12:14:11 web3 kernel:  [] :jbd:journal_stop+0x1cf/0x1ff
Jul 16 12:14:13 web3 nrpe[20013]: Error: Could not complete SSL handshake. 5
Jul 16 12:17:34 web3 nrpe[20026]: Error: Could not complete SSL handshake. 5
Jul 16 12:18:12 web3 kernel:  [] __writeback_single_inode+0x1e9/0x328
Jul 16 12:18:12 web3 kernel:  [] write_inode_now+0x77/0xbf
Jul 16 12:18:12 web3 kernel:  [] :nfsd:nfsd_setattr+0x3f9/0x426
Jul 16 12:18:12 web3 kernel:  [] :nfsd:nfsd3_proc_setattr+0x98/0xa4
Jul 16 12:18:12 web3 kernel:  [] :nfsd:nfsd_dispatch+0xd8/0x1d6
Jul 16 12:18:12 web3 kernel:  [] :sunrpc:svc_process+0x454/0x71b
Jul 16 12:18:13 web3 kernel:  [] __down_read+0x12/0x92
Jul 16 12:18:13 web3 kernel:  [] :nfsd:nfsd+0x0/0x2cb
Jul 16 12:18:13 web3 kernel:  [] :nfsd:nfsd+0x1a5/0x2cb
Jul 16 12:18:13 web3 kernel:  [] child_rip+0xa/0x11
Jul 16 12:18:13 web3 kernel:  [] :nfsd:nfsd+0x0/0x2cb
Jul 16 12:18:13 web3 kernel:  [] :nfsd:nfsd+0x0/0x2cb
Jul 16 12:18:13 web3 kernel:  [] child_rip+0x0/0x11
Jul 16 12:18:13 web3 kernel:
Jul 16 12:18:13 web3 kernel: INFO: task php-cgi:24520 blocked for more than 120 seconds.
Jul 16 12:18:13 web3 kernel: “echo 0 > /proc/sys/kernel/hung_task_timeout_secs” disables this message.
Jul 16 12:18:13 web3 kernel: php-cgi       D ffff8103da547ac0     0 24520  24516         24521 24519 (NOTLB)
Jul 16 12:18:13 web3 kernel:  ffff8102e056dd28 0000000000000086 0000000000004040 00000000000001ff
Jul 16 12:18:13 web3 kernel:  ffff8102e056dee8 000000000000000a ffff81023ed32040 ffff81020b7a07e0
Jul 16 12:18:13 web3 kernel:  0011e5a8d1911bbe 00000000000056f8 ffff81023ed32228 000000068003055b
Jul 16 12:18:13 web3 kernel: Call Trace:
Jul 16 12:18:13 web3 kernel:  [] __mutex_lock_slowpath+0x60/0x9b
Jul 16 12:18:13 web3 kernel:  [] .text.lock.mutex+0xf/0x14
Jul 16 12:18:13 web3 kernel:  [] generic_file_aio_write+0x4e/0xc1
Jul 16 12:18:13 web3 kernel:  [] :ext3:ext3_file_write+0x16/0x91
Jul 16 12:18:13 web3 kernel:  [] do_sync_write+0xc7/0x104
Jul 16 12:18:13 web3 kernel:  [] sys_recvfrom+0xd4/0x130
Jul 16 12:18:13 web3 kernel:  [] autoremove_wake_function+0x0/0x2e
Jul 16 12:18:13 web3 kernel:  [] _atomic_dec_and_lock+0x39/0x57
Jul 16 12:18:13 web3 kernel:  [] dput+0x3d/0x114
Jul 16 12:18:14 web3 kernel:  [] vfs_write+0xce/0x174
Jul 16 12:18:14 web3 kernel:  [] sys_write+0x45/0x6e
Jul 16 12:18:14 web3 kernel:  [] system_call+0x7e/0x83


INFO: task nfsd:3269 blocked for more than 120 seconds.
INFO: task php-cgi:24520 blocked for more than 120 seconds.
```
这句的意思是nfsd和php-cgi这两个进程出现了超过120秒的阻塞现象。初步怀疑是机房的网络出现了短暂中断或者磁盘写入出现问题造成的，只是日志又正常写进去了，网络的原因可能性比较大。有待后续观察…..