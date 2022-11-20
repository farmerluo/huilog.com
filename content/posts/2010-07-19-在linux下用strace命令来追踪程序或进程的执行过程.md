---
title: 在linux下用strace命令来追踪程序或进程的执行过程
author: 阿辉
date: 2010-07-19T10:19:00+00:00
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
在linux下有一个strace命令，可以用来追踪程序或进程的执行过程，从中查找和追踪程序的bug,及运行中的瓶颈等。

命令用法，主要有两种方式：

1. strace  程序

strace会运行这个程序，并追踪。

2. `strace -p pid`

这是追踪一个已经运行的程序。

另外还有一些参数也很有用，如-c可以生成一个统计结果，-o file可以把追踪信息输出到一个文件内。

<!--more-->

一个例子：
```bash
[root@web3 ~]# strace -p 1407
Process 1407 attached - interrupt to quit
accept(0, {sa_family=AF_INET, sin_port=htons(62532), sin_addr=inet_addr(“192.168.1.3”)}, [227633266704]) = 4
clock_gettime(CLOCK_MONOTONIC, {5251345, 650771219}) = 0
poll([{fd=4, events=POLLIN}], 1, 5000)  = 1 ([{fd=4, revents=POLLIN}])
read(4, “11110”, 8)         = 8
read(4, “1”, 8)          = 8
read(4, “141r!7”, 8)           = 8
read(4, “fQUERY_STRING164REQUEST_METHODPO”…, 3368) = 3368
read(4, “141”, 8)          = 8
clock_gettime(CLOCK_MONOTONIC, {5251345, 650953219}) = 0
rt_sigaction(SIGCHLD, {0x69f490, [CHLD], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x69f490, [CHLD], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
read(4, “151t7”, 8)          = 8
read(4, “act=start”, 9)                 = 9
read(4, “”, 7)            = 7
setitimer(ITIMER_PROF, {it_interval={0, 0}, it_value={60, 0}}, NULL) = 0
rt_sigaction(SIGPROF, {0x6d25a0, [PROF], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x6d25a0, [PROF], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigprocmask(SIG_UNBLOCK, [PROF], NULL, 8) = 0
rt_sigaction(SIGSEGV, {0x2abfe48f1250, [SEGV], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {SIG_DFL, [SEGV], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGFPE, {0x2abfe48f1250, [FPE], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {SIG_DFL, [FPE], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGBUS, {0x2abfe48f1250, [BUS], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {SIG_DFL, [BUS], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGILL, {0x2abfe48f1250, [ILL], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {SIG_DFL, [ILL], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGABRT, {0x2abfe48f1250, [ABRT], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {SIG_DFL, [ABRT], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
open(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/ajax.php”, O_RDONLY) = 14
fstat(14, {st_mode=S_IFREG|0644, st_size=3874, …}) = 0
lstat(“/home”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com”, {st_mode=S_IFDIR|0775, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/ajax.php”, {st_mode=S_IFREG|0644, st_size=3874, …}) = 0
chdir(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle”) = 0
fstat(14, {st_mode=S_IFREG|0644, st_size=3874, …}) = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x2abff02ce000
read(14, “<?phprndefine(‘IS_AJAX’, true);r”…, 4096) = 3874
lseek(14, 0, SEEK_SET)                  = 0
clock_gettime(CLOCK_MONOTONIC, {5251345, 651628219}) = 0
setitimer(ITIMER_PROF, {it_interval={0, 0}, it_value={30, 0}}, NULL) = 0
rt_sigaction(SIGPROF, {0x6d25a0, [PROF], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x6d25a0, [PROF], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigprocmask(SIG_UNBLOCK, [PROF], NULL, 8) = 0
fstat(14, {st_mode=S_IFREG|0644, st_size=3874, …}) = 0
close(14)                               = 0
munmap(0x2abff02ce000, 4096)            = 0
stat(“./init.php”, {st_mode=S_IFREG|0644, st_size=2433, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/init.php”, {st_mode=S_IFREG|0644, st_size=2433, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/PhpMapper.class.php”, {st_mode=S_IFREG|0644, st_size=12463, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/PhpMapper.class.php”, {st_mode=S_IFREG|0644, st_size=12463, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper//IDB.php”, {st_mode=S_IFREG|0644, st_size=318, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/IDB.php”, {st_mode=S_IFREG|0644, st_size=318, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/ICache.php”, {st_mode=S_IFREG|0644, st_size=335, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/ICache.php”, {st_mode=S_IFREG|0644, st_size=335, …}) = 0
lstat(“/home”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com”, {st_mode=S_IFDIR|0775, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/cache”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/cache/MemcacheDrive.class.php”, {st_mode=S_IFREG|0644, st_size=1870, …}) = 0
open(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/cache/MemcacheDrive.class.php”, O_RDONLY) = 14
fstat(14, {st_mode=S_IFREG|0644, st_size=1870, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/cache/MemcacheDrive.class.php”, {st_mode=S_IFREG|0644, st_size=1870, …}) = 0
close(14)                               = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPRemoting/PHPRemoting.php”, {st_mode=S_IFREG|0644, st_size=2806, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPRemoting/PHPRemoting.php”, {st_mode=S_IFREG|0644, st_size=2806, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPRemoting/Process/IRemotingProcess.php”, {st_mode=S_IFREG|0644, st_size=324, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPRemoting/Process/IRemotingProcess.php”, {st_mode=S_IFREG|0644, st_size=324, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/define.php”, {st_mode=S_IFREG|0644, st_size=2071, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/define.php”, {st_mode=S_IFREG|0644, st_size=2071, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/config.php”, {st_mode=S_IFREG|0644, st_size=860, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/config.php”, {st_mode=S_IFREG|0644, st_size=860, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/dns.php”, {st_mode=S_IFREG|0644, st_size=1040, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/dns.php”, {st_mode=S_IFREG|0644, st_size=1040, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/function.php”, {st_mode=S_IFREG|0644, st_size=7069, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/function.php”, {st_mode=S_IFREG|0644, st_size=7069, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/business/common.php”, {st_mode=S_IFREG|0644, st_size=2044, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/business/common.php”, {st_mode=S_IFREG|0644, st_size=2044, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/CSession.php”, {st_mode=S_IFREG|0644, st_size=377, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/CSession.php”, {st_mode=S_IFREG|0644, st_size=377, …}) = 0
socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 14
fcntl(14, F_GETFL)                      = 0x2 (flags O_RDWR)
fcntl(14, F_SETFL, O_RDWR|O_NONBLOCK)   = 0
connect(14, {sa_family=AF_INET, sin_port=htons(11211), sin_addr=inet_addr(“192.168.1.3”)}, 16) = -1 EINPROGRESS (Operation now in progress)
poll([{fd=14, events=POLLIN|POLLOUT|POLLERR|POLLHUP}], 1, -996660526) = 1 ([{fd=14, revents=POLLOUT}])
getsockopt(14, SOL_SOCKET, SO_ERROR, [-984040126352982016], [4]) = 0
fcntl(14, F_SETFL, O_RDWR)              = 0
sendto(14, “get 38ijluah285g6t6g43eha4r420rn”, 32, MSG_DONTWAIT, NULL, 0) = 32
poll([{fd=14, events=POLLIN|POLLERR|POLLHUP}], 1, -996660526) = 1 ([{fd=14, revents=POLLIN}])
recvfrom(14, “VALUE 38ijluah285g6t6g43eha4r420”…, 8192, MSG_DONTWAIT, NULL, NULL) = 75
lstat(“/home”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com”, {st_mode=S_IFDIR|0775, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/langs”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/langs/en”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/langs/en/lang.ini”, {st_mode=S_IFREG|0644, st_size=30, …}) = 0
open(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/langs/en/lang.ini”, O_RDONLY) = 16
fstat(16, {st_mode=S_IFREG|0644, st_size=30, …}) = 0
read(16, “[common]rnkey=sdfksafasdfjk %s”, 8192) = 30
read(16, “”, 8192)                      = 0
read(16, “”, 8192)                      = 0
close(16)                               = 0
lstat(“/home”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com”, {st_mode=S_IFDIR|0775, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/drives”, {st_mode=S_IFDIR|0755, st_size=4096, …}) = 0
lstat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/drives/MssqlDrive.class.php”, {st_mode=S_IFREG|0644, st_size=1557, …}) = 0
open(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/drives/MssqlDrive.class.php”, O_RDONLY) = 16
fstat(16, {st_mode=S_IFREG|0644, st_size=1557, …}) = 0
stat(“/home/httpd/funtime.shinezone.com/web/bejeweledpuzzle/include/PHPMapper/drives/MssqlDrive.class.php”, {st_mode=S_IFREG|0644, st_size=1557, …}) = 0
read(16, “<?phpn/*n * SQLServer 351251261345212250346224257”…, 8192) = 1557
read(16, “”, 8192)                      = 0
read(16, “”, 8192)                      = 0
close(16)                               = 0
poll([{fd=15, events=POLLOUT}], 1, 1000) = 1 ([{fd=15, revents=POLLOUT}])
sendto(15, “11361use gamelib”, 30, MSG_NOSIGNAL, NULL, 0) = 30
poll([{fd=15, events=POLLIN}], 1, 1000) = 1 ([{fd=15, revents=POLLIN}])
recvfrom(15, “4125320s1”, 8, MSG_NOSIGNAL, NULL, NULL) = 8
poll([{fd=15, events=POLLIN}], 1, 1000) = 1 ([{fd=15, revents=POLLIN}])
recvfrom(15, “3433717Gamelib7Gameli”…, 163, MSG_NOSIGNAL, NULL, NULL) = 163
poll([{fd=15, events=POLLOUT}], 1, 1000) = 1 ([{fd=15, revents=POLLOUT}])
sendto(15, “11L1SELECT * FRO”…, 76, MSG_NOSIGNAL, NULL, 0) = 76
poll([{fd=15, events=POLLIN}], 1, 1000) = 1 ([{fd=15, revents=POLLIN}])
recvfrom(15, “4123120s1”, 8, MSG_NOSIGNAL, NULL, NULL) = 8
poll([{fd=15, events=POLLIN}], 1, 1000) = 1 ([{fd=15, revents=POLLIN}])
recvfrom(15, “20117200083gIdt247244103205gN”…, 529, MSG_NOSIGNAL, NULL, NULL) = 529
poll([{fd=15, events=POLLOUT}], 1, 1000) = 1 ([{fd=15, revents=POLLOUT}])
sendto(15, “11r1UPDATE gl_ga”…, 114, MSG_NOSIGNAL, NULL, 0) = 114
poll([{fd=15, events=POLLIN}], 1, 1000) = 1 ([{fd=15, revents=POLLIN}])
recvfrom(15, “412120s1”, 8, MSG_NOSIGNAL, NULL, NULL) = 8
poll([{fd=15, events=POLLIN}], 1, 1000) = 1 ([{fd=15, revents=POLLIN}])
recvfrom(15, “375203051”, 9, MSG_NOSIGNAL, NULL, NULL) = 9
rt_sigaction(SIGSEGV, {SIG_DFL, [SEGV], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x2abfe48f1250, [SEGV], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGFPE, {SIG_DFL, [FPE], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x2abfe48f1250, [FPE], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGBUS, {SIG_DFL, [BUS], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x2abfe48f1250, [BUS], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGILL, {SIG_DFL, [ILL], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x2abfe48f1250, [ILL], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
rt_sigaction(SIGABRT, {SIG_DFL, [ABRT], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, {0x2abfe48f1250, [ABRT], SA_RESTORER|SA_RESTART, 0x3c4c8302d0}, 8) = 0
sendto(14, “set 38ijluah285g6t6g43eha4r420 0”…, 75, MSG_DONTWAIT, NULL, 0) = 75
poll([{fd=14, events=POLLIN|POLLERR|POLLHUP}], 1, -996660526) = 1 ([{fd=14, revents=POLLIN}])
recvfrom(14, “STOREDrn”, 8192, MSG_DONTWAIT, NULL, NULL) = 8
close(14)                               = 0
write(4, “1613444X-Powered-By: PHP/5.2.11”…, 240) = 240
setitimer(ITIMER_PROF, {it_interval={0, 0}, it_value={0, 0}}, NULL) = 0
write(4, “13110ere”, 16) = 16
shutdown(4, 1 / send */)               = 0
recvfrom(4, “151”, 8, 0, NULL, NULL) = 8
recvfrom(4, “”, 8, 0, NULL, NULL)       = 0
close(4)                                = 0
clock_gettime(CLOCK_MONOTONIC, {5251345, 659076219}) = 0
clock_gettime(CLOCK_MONOTONIC, {5251345, 659137219}) = 0
accept(0,  <unfinished …>
Process 1407 detached
```