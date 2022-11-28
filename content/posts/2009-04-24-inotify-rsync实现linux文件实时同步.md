---
title: inotify + rsync实现linux文件实时同步
author: 阿辉
date: 2009-04-24T10:39:00+00:00
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
公司一套系统的同步使用的donotify，不能实现子目录的实时同步，通过查资料，发现inotify可以实现子目录的实时同步，以下为笔记。

一、介绍Inotify 是文件系统事件监控机制，作为 dnotify 的有效替代。dnotify 是较早内核支持的文件监控机制。Inotify 是一种强大的、细粒度的、异步的机制，它满足各种各样的文件监控需要，不仅限于安全和性能。

inotify 可以监视的文件系统事件包括：IN_ACCESS，即文件被访问IN_MODIFY，文件被 writeIN_ATTRIB，文件属性被修改，如 chmod、chown、touch 等IN_CLOSE_WRITE，可写文件被 closeIN_CLOSE_NOWRITE，不可写文件被 closeIN_OPEN，文件被 openIN_MOVED_FROM，文件被移走,如 mvIN_MOVED_TO，文件被移来，如 mv、cpIN_CREATE，创建新文件IN_DELETE，文件被删除，如 rmIN_DELETE_SELF，自删除，即一个可执行文件在执行时删除自己IN_MOVE_SELF，自移动，即一个可执行文件在执行时移动自己IN_UNMOUNT，宿主文件系统被 umountIN_CLOSE，文件被关闭，等同于(IN_CLOSE_WRITE | IN_CLOSE_NOWRITE)IN_MOVE，文件被移动，等同于(IN_MOVED_FROM | IN_MOVED_TO)注：上面所说的文件也包括目录。

<!--more-->

二、为能在shell下使用inotify特性，需要安装inotify-tools

1、inotify-tools：The general purpose of this package is to allow inotify’s features to be used from within shell scripts.

下载地址：http://inotify-tools.sourceforge.net/

编译安装`./configuremakemake install`完成后，注意查看`manpage，man inotify 、 man inotifywait`

inotifywait 仅执行阻塞，等待 inotify 事件。您可以监控任何一组文件和目录，或监控整个目录树（目录、子目录、子目录的子目录等等）。在 shell 脚本中使用 inotifywait。

inotifywatch 收集关于被监视的文件系统的统计数据，包括每个 inotify 事件发生多少次。

2、inotify的系统相关参数： 
```
   /proc interfaces   The following interfaces can be used to limit the amount of kernel memory consumed by inotify:
   /proc/sys/fs/inotify/max_queued_events   The value in this file is used when an application calls inotify_init(2) to set an upper limit on the number of events that can be queued to the corresponding inotify instance. Events in excess of this limit are dropped, but an IN_Q_OVERFLOW event is always generated.
   /proc/sys/fs/inotify/max_user_instances   This specifies an upper limit the number of inotify instances that can be created per real user ID.
   /proc/sys/fs/inotify/max_user_watches   This specifies a limit the number of watches that can be associated with each inotify instance.
```

3、inotifywait 相关的参数（更多，查看manpage）：
```
inotifywait This command simply blocks for inotify events, making it appropriate for use in shell scripts. It can watch any set of files and directories, and can recursively watch entire directory trees.
-m, –monitor   Instead of exiting after receiving a single event, execute indefinitely. The default behaviour is to exit after the first event occurs.
-r, –recursive   Watch all subdirectories of any directories passed as arguments. Watches will be set up recursively to an unlimited depth. Symbolic links are not
traversed. Newly created subdirectories will also be watched.
-q, –quiet   If specified ce, the program will be less verbose. Specifically, it will not state when it has completed establishing all inotify watches. 
-e , –event    Listen for specific event(s) ly. The events which can be listened for are listed in the EVENTS section. This option can be specified more than ce. If omitted, all events are listened for. use“，”separate multi events
```

三、使用

1.查看是否支持inotify，从kernel 2.6.13开始正式并入内核，RHEL5已经支持。看看是否有 /proc/sys/fs/inotify/目录，以确定内核是否支持inotify
```
[root@RHEL5 Rsync]# ll /proc/sys/fs/inotify
total 0
-rw-r–r– 1 root root 0 Oct 9 09:36 max_queued_events
-rw-r–r– 1 root root 0 Oct 9 09:36 max_user_instances
-rw-r–r– 1 root root 0 Oct 9 09:36 max_user_watches
```

2.关于递归：inotifywait

This command simply blocks for inotify events, making it appropriate for use in shell scripts. It can watch any set of files and directories, and can recursively watch entire directory trees.

3.使用：
```bash
#!/bin/sh
src=/opt/webmail
des=/tmp
ip=192.168.7.192
/usr/local/bin/inotifywait -mrq –timefmt ‘%d/%m/%y %H:%M’ –format ‘%T %w%f’ -e modify,delete,create,attrib {src} | while read file   
do   
    rsync -avz –delete –progress {src} root@{ip}:{des} &&   echo “{file} was rsynced”  
    echo “—————————————————————————“
done
```
注：
当要排出同步某个目录时，为rsync添加–exculde=PATTERN参数，注意，路径是相对路径。详细查看man rsync

当要排除都某个目录的事件监控的处理时，为inotifywait添加–exclude或–excludei参数。详细查看man inotifywait

另：`/usr/local/bin/inotifywait -mrq –timefmt ‘%d/%m/%y %H:%M’ –format ‘%T %w%f’ -e modify,delete,create,attrib {src}` 上面的命令返回的值类似于：`10/03/09 15:31 /wwwpic/1`这3个返回值做为参数传给read，关于此处，有人是这样写的：`inotifywait -mrq -e create,move,delete,modify $SRC | while read D E F;do`
细化了返回值。

说明： 当文件系统发现指定目录下有如上的条件的时候就触发相应的指令，是一种主动告之的而非我用循环比较目录下的文件的异动，该程序在运行时，更改目录内的文件时系统内核会发送一个信号，这个信号会触发运行rsync命令，这时会同步源目录和目标目录。

–timefmt：指定输出时的输出格式   

–format： ‘%T %w%f’指定输出的格式，上面的输出类似于：`12/10/08 06:34 /opt/webmail/dovecot-1.1.2/src/test/1`