---
title: Redis 在Centos Linux 5.x上的启动脚本
author: 阿辉
date: 2011-05-20T11:08:00+00:00
categories:
- Redis
tags:
- Redis
keywords:
- Redis
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
发现网上的Redis管理脚本基于Ubuntu 的发行版上的，在Centos linux 5.x上并不能用，所以自己写了一个。

用这个脚本管理之前，需要先配置下面的内核参数，否则Redis脚本在重启或停止redis时，将会报错，并且不能自动在停止服务前同步数据到磁盘上：
```
# vim /etc/sysctl.conf
vm.overcommit_memory = 1
```
然后应用生效：
```
# sysctl -p
```
<!--more-->

建立redis启动脚本：
```bash
# vim /etc/init.d/redis

#!/bin/bash
#
# Init file for redis
#
# chkconfig: - 80 12
# description: redis daemon
#
# processname: redis
# config: /etc/redis.conf
# pidfile: /var/run/redis.pid

source /etc/init.d/functions

#BIN="/usr/local/bin"
BIN="/usr/local/bin"
CONFIG="/etc/redis.conf"
PIDFILE="/var/run/redis.pid"


### Read configuration
[ -r "$SYSCONFIG" ] && source "$SYSCONFIG"

RETVAL=0
prog="redis-server"
desc="Redis Server"

start() {

        if [ -e $PIDFILE ];then
             echo "$desc already running...."
             exit 1
        fi

        echo -n $"Starting $desc: "
        daemon $BIN/$prog $CONFIG

        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && touch /var/lock/subsys/$prog
        return $RETVAL
}

stop() {
        echo -n $"Stop $desc: "
        killproc $prog
        RETVAL=$?
        echo
        [ $RETVAL -eq 0 ] && rm -f /var/lock/subsys/$prog $PIDFILE
        return $RETVAL
}

restart() {
        stop
        start
}


case "$1" in
  start)
        start
        ;;
  stop)
        stop
        ;;
  restart)
        restart
        ;;
  condrestart)
        [ -e /var/lock/subsys/$prog ] && restart
        RETVAL=$?
        ;;
  status)
        status $prog
        RETVAL=$?
        ;;
   *)
        echo $"Usage: $0 {start|stop|restart|condrestart|status}"
        RETVAL=1
esac

exit $RETVAL
```

然后增加服务并开机自启动：
```bash
# chmod 755 /etc/init.d/redis
# chkconfig --add redis
# chkconfig --level 345 redis on
# chkconfig --list redis
```