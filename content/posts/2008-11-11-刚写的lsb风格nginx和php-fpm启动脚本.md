---
title: 刚写的lsb风格nginx和php-fpm启动脚本
author: 阿辉
date: 2008-11-11T16:32:00+00:00
categories:
- Nginx
tags:
- Nginx
keywords:
- Nginx
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
工作需要，把之前jackbillow写的nginx的启动脚本改了下。改成nginx和php-fpm一起管理的。完成lsb风格，看起来舒服。

注意：nginx及php-fpm的pid文件不能变。

2009.08.21:增加ulimit，设置文件最大打开数

<!--more-->

```bash
#!/bin/bash
# nginx and php_fpm Startup script for the Nginx HTTP Server
# this script create it by Luo Hui at 2008.11.11.
# The changes in jackbillow’s nginx startup script v.0.0.2.
# it is v.0.0.3 version.
# if you find any errors on this scripts,please contact Luo Hui.
# and send mail to luohui at fortelchina dot com.
#
# chkconfig: - 85 15
# description: Nginx is a high-performance web and proxy server.
#              Php-fpm is a php-cgi service.
# processname: nginxphp
# nginx pidfile: /var/run/nginx.pid
# nginx config: /usr/local/nginx/conf/nginx.conf
# php-fpm pidfile: /var/run/php-cgi.pid
# php-fpm config: /usr/local/php5/etc/php-fpm.conf

nginxd=/usr/local/nginx/sbin/nginx
nginx_config=/usr/local/nginx/conf/nginx.conf
nginx_pid=/var/run/nginx.pid

php_fpm_BIN=/usr/local/php5/bin/php-cgi
php_fpm_CONF=/usr/local/php5/etc/php-fpm.conf
php_fpm_PID=/var/run/php-cgi.pid
php_opts=”–fpm-config php_fpm_CONF”

RETVAL=0
prog=”nginx”

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ {NETWORKING} = “no” ] && exit 0

[ -x nginxd ] || exit 0
[ -x php_fpm_BIN ] || exit 0

ulimit -HSn 65535

wait_for_pid () {
try=0  

while test try -lt 35 ; do

case “1” in
‘created’)
if [ -f “2” ] ; then
try=’’
break
fi
;;

‘removed’)
if [ ! -f “2” ] ; then
try=’’
break
fi
;;
esac

#                echo -n .
try=expr try + 1
sleep 1

done

}

php_fpm_start() {
echo -n “Starting php_fpm: “

php_fpm_BIN –fpm php_opts && success || failure
RETVAL=?
echo

if [ “RETVAL” != 0 ] ; then
return RETVAL
fi

wait_for_pid created php_fpm_PID

if [ -n “try” ] ; then
RETVAL=1
else
RETVAL=0
fi

return RETVAL
}

php_fpm_stop() {

echo -n “Stopping php_fpm: “

if [ ! -r php_fpm_PID ] ; then
echo “warning, no pid file found - php-fpm is not running ?”
exit 1
fi

#kill -TERM cat php_fpm_PID
killproc php_fpm_BIN -TERM

wait_for_pid removed php_fpm_PID

if [ -n “try” ] ; then
RETVAL=1
failure
else
RETVAL=0
success
fi
echo

[ RETVAL = 0 ] && rm -f /var/lock/subsys/php-cgi php_fpm_PID
return RETVAL
}

php_fpm_reload() {
echo -n “Reloading php-fpm: “

if [ ! -r php_fpm_PID ] ; then
echo “warning, no pid file found - php-fpm is not running ?”
exit 1
fi

#    kill -USR2 cat php_fpm_PID
killproc php_fpm_BIN -USR2
RETVAL=?
echo

}



# Start nginx daemons functions.
nginx_start() {

if [ -e nginx_pid ];then
echo “nginx already running….”
exit 1
fi

echo -n ”Starting prog: “
daemon nginxd -c {nginx_config}
RETVAL=?
echo
[ RETVAL = 0 ] && touch /var/lock/subsys/nginx
return RETVAL

}


# Stop nginx daemons functions.
nginx_stop() {
echo -n ”Stopping prog: “
killproc nginxd
RETVAL=?
echo
[ RETVAL = 0 ] && rm -f /var/lock/subsys/nginx nginx_pid
}


# reload nginx service functions.
nginx_reload() {

echo -n ”Reloading prog: “
#kill -HUP cat {nginx_pid}
killproc nginxd -HUP
RETVAL=?
echo

}

# See how we were called.
case “1” in
start)
php_fpm_start
nginx_start
;;

stop)
nginx_stop
php_fpm_stop
;;

reload)
nginx_reload
php_fpm_reload
;;

restart)
nginx_stop
php_fpm_stop
nginx_start
php_fpm_start
;;

status)
status “php-cgi”
RETVAL=?
status prog
RETVAL=?
;;
*)
echo ”Usage: nginxphp {start|stop|restart|reload|status|help}”
exit 1
esac

exit RETVAL
```