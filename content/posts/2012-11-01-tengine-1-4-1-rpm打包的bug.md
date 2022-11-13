---
title: tengine 1.4.1 rpm打包的bug
author: 阿辉
date: 2012-11-01T02:47:09+00:00
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
tengine对limit_req模块进行了改进，可以同时对几个条件进行并发限制，在实际情况下，会比较实用，因为nginx自带的limit_req只能对一个条件进行限制，如果只对IP进行限制，会误伤很多正常用户；而tengine的limit_req模板支持多条件限制并发，配置成url加IP的模块限制并发，基本上误伤很小了。

我们一般在正式环境都是打成rpm包的方式用puppet进行批量部署，发现tengine有个小bug。下面是我的spec文件：

<!--more-->

```bash
%define _prefix         /data1/app/services/tengine
%define _user           www
%define _user_uid       600
%define _group          www
%define _group_gid      600
%define _sbin_path      /usr/sbin

%define name      tengine
%define summary   tengine for shinezone
%define version   1.4.1
%define release   1
%define license   GPL
%define group     shinezone/software
%define source    tengine-%{version}.tar.gz
%define url       http://tengine.taobao.org/
%define vendor    shinezone
%define packager  farmer.luo

Name:      %{name}
Version:   %{version}
Release:   %{release}
Packager:  %{packager}
Vendor:    %{vendor}
License:   %{license}
Summary:   %{summary}
Group:     %{group}
Source:    %{source}
URL:       %{url}
Prefix:    %{_prefix}
Buildroot: %{buildroot}

BuildRequires:  pcre-devel
BuildRequires:  zlib-devel
BuildRequires:  mhash-devel
BuildRequires:  openssl-devel
BuildRequires:  libxml2-devel 
BuildRequires:  libxslt-devel 
BuildRequires:  gd-devel 
BuildRequires:  lua-devel 
BuildRequires:  geoip-devel

Requires: pcre
Requires: zlib
Requires: mhash
Requires: openssl
Requires: libxml2
Requires: libxslt
Requires: gd
Requires: lua
Requires: geoip

%description
tengine for shinezone

%prep
%setup -q -n tengine-%{version}

%build

./configure 
--prefix=%{_prefix} 
--user=%{_user} 
--group=%{_group} 
--with-poll_module 
--with-pcre 
--with-http_realip_module 
--with-http_dav_module 
--with-http_gzip_static_module 
--with-http_degradation_module 
--with-http_addition_module=shared 
--with-http_xslt_module=shared 
--with-http_image_filter_module=shared 
--with-http_geoip_module=shared 
--with-http_sub_module=shared 
--with-http_flv_module=shared 
--with-http_slice_module=shared 
--with-http_mp4_module=shared 
--with-http_concat_module=shared 
--with-http_random_index_module=shared 
--with-http_secure_link_module=shared 
--with-http_sysguard_module=shared 
--with-http_charset_filter_module=shared 
--with-http_userid_filter_module=shared 
--with-http_footer_filter_module=shared 
--with-http_access_module=shared 
--with-http_autoindex_module=shared 
--with-http_map_module=shared 
--with-http_split_clients_module=shared 
--with-http_referer_module=shared 
--with-http_rewrite_module=shared 
--with-http_fastcgi_module=shared 
--with-http_uwsgi_module=shared 
--with-http_scgi_module=shared 
--with-http_memcached_module=shared 
--with-http_limit_conn_module=shared 
--with-http_limit_req_module=shared 
--with-http_empty_gif_module=shared 
--with-http_browser_module=shared 
--with-http_user_agent_module=shared 
--with-http_upstream_ip_hash_module=shared 
--with-http_upstream_least_conn_module=shared 
--with-http_lua_module

make

%install
rm -rf %{buildroot}
make install DESTDIR=%{buildroot}
make dso_install DESTDIR=%{buildroot}

mkdir -p %{buildroot}/%{_initrddir}
(
cat <<'EOF'
#!/bin/bash
# tengine Startup script for the tengine HTTP Server
# this script create it by Luo Hui at 2008.11.11.
# if you find any errors on this scripts,please contact Luo Hui.
# and send mail to farmer.luo at gmail dot com.
#
# chkconfig: - 85 15
# description: tengine is a high-performance web and proxy server.
# processname: tengine
# tengine pidfile: /var/run/tengine.pid
# tengine config: /usr/local/tengine/conf/nginx.conf


nginxd=%{_prefix}/sbin/nginx
nginx_config=%{_prefix}/conf/nginx.conf
nginx_pid=/var/run/tengine.pid

RETVAL=0
prog="nginx"

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ ${NETWORKING} = "no" ] && exit 0

[ -x $nginxd ] || exit 0

ulimit -HSn 65535


# Start tengine daemons functions.
nginx_start() {

        if [ -e $nginx_pid ];then
                echo "tengine already running...."
                exit 1
        fi

        if [ ! -d %{_prefix}/logs ];then
                mkdir -p %{_prefix}/logs
        fi

        if [ ! -d %{_prefix}/tmp ]; then
                mkdir -p %{_prefix}/tmp
        fi

        if [ -e $nginx_pid ];then
                echo "tengine already running...."
                exit 1
        fi

        echo -n $"Starting $prog: "
        daemon $nginxd -c ${nginx_config}
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && touch /var/lock/subsys/tengine
        return $RETVAL

}


# Stop tengine daemons functions.
nginx_stop() {
        echo -n $"Stopping $prog: "
        killproc $nginxd
        RETVAL=$?
        echo
        [ $RETVAL = 0 ] && rm -f /var/lock/subsys/tengine $nginx_pid
}


# reload tengine service functions.
nginx_reload() {

        echo -n $"Reloading $prog: "
        #kill -HUP `cat ${nginx_pid}`
        killproc $nginxd -HUP
        RETVAL=$?
        echo

}

# See how we were called.
case "$1" in
start)
        nginx_start
        ;;

stop)
        nginx_stop
        ;;

reload)
        nginx_reload
        ;;

restart)
        nginx_stop
        nginx_start
        ;;

status)
        status $prog
        RETVAL=$?
        ;;
*)
        echo $"Usage: tengine {start|stop|restart|reload|status|help}"
        exit 1
esac

exit $RETVAL
EOF
) >%{buildroot}/%{_initrddir}/tengine

chmod 755 %{buildroot}/%{_initrddir}/tengine


%clean
rm -rf %{buildroot}


%pre
grep -q ^%{_group}: /etc/group || %{_sbin_path}/groupadd -g %{_group_gid} %{_group}
grep -q ^%{_user}: /etc/passwd || %{_sbin_path}/useradd -g %{_group} -u %{_user_uid} -d %{_prefix} -s /sbin/nologin -M %{_user}


%post
chkconfig --add tengine
chkconfig --level 345 tengine on


%preun
chkconfig --del tengine


%postun
if [ $1 = 0 ]; then
        userdel %{_user} > /dev/null 2>&1 || true
fi

%files
%defattr(-,root,root)
%dir %{_prefix}/
%attr(0755,%{_user},%{_group}) %dir %{_prefix}/logs
#%attr(0700,%{_user},%{_group}) %dir %{_prefix}/client_body_temp
#%attr(0700,%{_user},%{_group}) %dir %{_prefix}/fastcgi_temp
#%attr(0700,%{_user},%{_group}) %dir %{_prefix}/proxy_temp
#%attr(0700,%{_user},%{_group}) %dir %{_prefix}/scgi_temp
#%attr(0700,%{_user},%{_group}) %dir %{_prefix}/uwsgi_temp
%dir %{_prefix}/modules
%dir %{_prefix}/sbin
%dir %{_prefix}/conf
%dir %{_prefix}/html
%{_prefix}/sbin/nginx
%{_prefix}/sbin/dso_tool
%{_prefix}/conf/module_stubs
%{_prefix}/conf/fastcgi.conf
%{_prefix}/conf/fastcgi_params.default
%{_prefix}/conf/win-utf
%{_prefix}/conf/koi-utf
%{_prefix}/conf/nginx.conf.default
%{_prefix}/conf/fastcgi.conf.default
%{_prefix}/conf/fastcgi_params
%{_prefix}/conf/koi-win
%{_prefix}/conf/mime.types
%{_prefix}/conf/nginx.conf
%{_prefix}/conf/mime.types.default
%{_prefix}/conf/scgi_params
%{_prefix}/conf/scgi_params.default
%{_prefix}/conf/uwsgi_params
%{_prefix}/conf/uwsgi_params.default
%{_prefix}/html/50x.html
%{_prefix}/html/index.html
%{_prefix}/modules/ngx_http_access_module.so
%{_prefix}/modules/ngx_http_addition_filter_module.so
%{_prefix}/modules/ngx_http_autoindex_module.so
%{_prefix}/modules/ngx_http_browser_module.so
%{_prefix}/modules/ngx_http_charset_filter_module.so
%{_prefix}/modules/ngx_http_concat_module.so
%{_prefix}/modules/ngx_http_empty_gif_module.so
%{_prefix}/modules/ngx_http_fastcgi_module.so
%{_prefix}/modules/ngx_http_flv_module.so
%{_prefix}/modules/ngx_http_footer_filter_module.so
%{_prefix}/modules/ngx_http_geoip_module.so
%{_prefix}/modules/ngx_http_image_filter_module.so
%{_prefix}/modules/ngx_http_limit_conn_module.so
%{_prefix}/modules/ngx_http_limit_req_module.so
%{_prefix}/modules/ngx_http_map_module.so
%{_prefix}/modules/ngx_http_memcached_module.so
%{_prefix}/modules/ngx_http_mp4_module.so
%{_prefix}/modules/ngx_http_random_index_module.so
%{_prefix}/modules/ngx_http_referer_module.so
%{_prefix}/modules/ngx_http_rewrite_module.so
%{_prefix}/modules/ngx_http_scgi_module.so
%{_prefix}/modules/ngx_http_secure_link_module.so
%{_prefix}/modules/ngx_http_slice_module.so
%{_prefix}/modules/ngx_http_split_clients_module.so
%{_prefix}/modules/ngx_http_sub_filter_module.so
%{_prefix}/modules/ngx_http_sysguard_module.so
%{_prefix}/modules/ngx_http_upstream_ip_hash_module.so
%{_prefix}/modules/ngx_http_upstream_least_conn_module.so
%{_prefix}/modules/ngx_http_user_agent_module.so
%{_prefix}/modules/ngx_http_userid_filter_module.so
%{_prefix}/modules/ngx_http_uwsgi_module.so
%{_prefix}/modules/ngx_http_xslt_filter_module.so
%{_initrddir}/tengine

%changelog
* Fri Oct 19 2012 Luo Hui <farmer dot luo at gmail.com>
- init

```

打包时会出错：

```bash
test -f '/root/rpmbuild/BUILDROOT/tengine-1.4.1-1.x86_64/data1/app/services/tengine/conf/module_stubs'          || cp objs/module_stubs '/root/rpmbuild/BUILDROOT/tengine-1.4.1-1.x86_64/data1/app/services/tengine/conf'
cp objs/module_stubs '/root/rpmbuild/BUILDROOT/tengine-1.4.1-1.x86_64/data1/app/services/tengine/conf/module_stubs'
chmod 0755 objs/dso_tool
cp objs/dso_tool '/data1/app/services/tengine/sbin'
cp: cannot create regular file `/data1/app/services/tengine/sbin': No such file or directory
make[1]: *** [install] Error 1
make[1]: Leaving directory `/root/rpmbuild/BUILD/tengine-1.4.1'
make: *** [install] Error 2
error: Bad exit status from /var/tmp/rpm-tmp.RjSntT (%install)

RPM build errors:
Bad exit status from /var/tmp/rpm-tmp.RjSntT (%install)

```

研究后发现，是因为Makefile内的make dso_install的不支持$(DESTDIR)造成的，按下面的方式改一下就好了。

`vim tengine-1.4.1/auto/install`

把241行的:

`cp $NGX_DSO_COMPILE '$NGX_PREFIX/sbin'`

改成：

`cp $NGX_DSO_COMPILE '$(DESTDIR)$NGX_PREFIX/sbin'`


然后把这个改过的tengine-1.4.1重新打成源码包，再rpmbuild就行了。

```bash

tar czf  tengine-1.4.1.tar.gz tengine-1.4.1

cp tengine-1.4.1.tar.gz rpmbuild/SOURCES/

rpmbuild -bb tengine.spec

```