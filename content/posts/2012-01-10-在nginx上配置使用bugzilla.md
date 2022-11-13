---
title: 在nginx上配置使用bugzilla
author: 阿辉
date: 2012-01-10T17:54:00+00:00
categories:
- Bugzilla
tags:
- Bugzilla
- Nginx
keywords:
- Bugzilla
- Nginx
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
配置bugzilla前需要先安装epel库和rpmforge库。

nginx本身不带perl解析的功能，需要用到fcgi-perl,安装：
```
yum install fcgi-perl
```
写一个脚本fastcgi服务的脚本：
<!--more-->

```perl
vim /usr/local/fcgi-perl/fastcgi-wrapper.pl

#!/usr/bin/perl

use FCGI;
use Socket;
use POSIX qw(setsid);

require ‘syscall.ph’;

&daemonize;

#this keeps the program alive or something after exec’ing perl scripts
END() { } BEGIN() { }
*CORE::GLOBAL::exit = sub { die "fakeexitnrc=".shift()."n"; };
eval q{exit};
if ($@) {
    exit unless $@ =~ /^fakeexit/;
};

&main;

sub daemonize() {
    chdir ‘/’                 or die "Can’t chdir to /: $!";
    defined(my $pid = fork)   or die "Can’t fork: $!";
    exit if $pid;
    setsid                    or die "Can’t start a new session: $!";
    umask 0;
}

sub main {
        $socket = FCGI::OpenSocket( "127.0.0.1:8999", 10 ); #use IP sockets
        $request = FCGI::Request( *STDIN, *STDOUT, *STDERR, %req_params, $socket );
        if ($request) { request_loop()};
            FCGI::CloseSocket( $socket );
}

sub request_loop {
        while( $request->Accept() >= 0 ) {

           #processing any STDIN input from WebServer (for CGI-POST actions)
           $stdin_passthrough =”;
           $req_len = 0 + $req_params{‘CONTENT_LENGTH’};
           if (($req_params{‘REQUEST_METHOD’} eq ‘POST’) && ($req_len != 0) ){
                my $bytes_read = 0;
                while ($bytes_read < $req_len) {
                        my $data = ”;
                        my $bytes = read(STDIN, $data, ($req_len – $bytes_read));
                        last if ($bytes == 0 || !defined($bytes));
                        $stdin_passthrough .= $data;
                        $bytes_read += $bytes;
                }
            }

            #running the cgi app
            if ( (-x $req_params{SCRIPT_FILENAME}) &&  #can I execute this?
                 (-s $req_params{SCRIPT_FILENAME}) &&  #Is this file empty?
                 (-r $req_params{SCRIPT_FILENAME})     #can I read this file?
            ){
        pipe(CHILD_RD, PARENT_WR);
        my $pid = open(KID_TO_READ, "-|");
        unless(defined($pid)) {
            print("Content-type: text/plainrnrn");
                        print "Error: CGI app returned no output – ";
                        print "Executing $req_params{SCRIPT_FILENAME} failed !n";
            next;
        }
        if ($pid > 0) {
            close(CHILD_RD);
            print PARENT_WR $stdin_passthrough;
            close(PARENT_WR);

            while(my $s = <KID_TO_READ>) { print $s; }
            close KID_TO_READ;
            waitpid($pid, 0);
        } else {
                    foreach $key ( keys %req_params){
                       $ENV{$key} = $req_params{$key};
                    }
                    # cd to the script’s local directory
                    if ($req_params{SCRIPT_FILENAME} =~ /^(.*)/[^/]+$/) {
                            chdir $1;
                    }

            close(PARENT_WR);
            close(STDIN);
            #fcntl(CHILD_RD, F_DUPFD, 0);
            syscall(&SYS_dup2, fileno(CHILD_RD), 0);
            #open(STDIN, "<&CHILD_RD");
            exec($req_params{SCRIPT_FILENAME});
            die("exec failed");
        }
            }
            else {
                print("Content-type: text/plainrnrn");
                print "Error: No such CGI app – $req_params{SCRIPT_FILENAME} may not ";
                print "exist or is not executable by this process.n";
            }

        }
}
```

做成服务运行：
```bash
vim /etc/init.d/perl-fastcgi

#!/bin/sh
#
# nginx – this script starts and stops the nginx daemon
#
# chkconfig: – 85 15
# description: Nginx is an HTTP(S) server, HTTP(S) reverse
# proxy and IMAP/POP3 proxy server
# processname: nginx
# config: /opt/nginx/conf/nginx.conf
# pidfile: /opt/nginx/logs/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

perlfastcgi="/usr/local/fcgi-perl/fastcgi-wrapper.pl"
prog=$(basename perl)

lockfile=/var/lock/subsys/perl-fastcgi

start() {
    [ -x $perlfastcgi ] || exit 5
    echo -n $"Starting $prog: "
    daemon $perlfastcgi
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    stop
    start
}

force_reload() {
    restart
}
rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart)
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
        ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|force-reload}"
        exit 2
    esac
```
 

改权限，并启动：
```bash
chmod 755 /etc/init.d/perl-fastcgi

chmod 755 /usr/local/fcgi-perl/fastcgi-wrapper.pl

chkconfig –add perl-fastcgi

chkconfig –level 345 perl-fastcgi on
service perl-fastcgi start
```

启动后可以用ps 或netstat看下进程和端口有没有起来。

nginx的安装就不讲了，在nginx内配置一个bugzilla使用的主机:
```
   server {
        listen       80;
        server_name  bugzilla.domainname.com;

        root   /home/httpd/bugzilla.domainname.com/web;
        index  index.cgi index.pl index.html index.htm;

        access_log /home/httpd/bugzilla.domainname.com/logs/access.log  combined;
        error_log  /home/httpd/bugzilla.domainname.com/logs/error.log notice;

#       location / {
#               root   /home/httpd/bugzilla.domainname.com/web;
#               index  index.html index.htm;
#       }

        location ~ .*.(pl|cgi)$ {
                gzip off;
                include fastcgi_params;
                fastcgi_pass  127.0.0.1:8999;
                fastcgi_index index.cgi;
                fastcgi_param  SCRIPT_FILENAME  /home/httpd/bugzilla.domainname.com/web$fastcgi_script_name;
        }

   }
```

应用配置：
```
service nginx reload
```

先写一个测试脚本跑跑看是否正常：

```perl
vim /home/httpd/bugzilla.domainname.com/web/test.pl 

#!/usr/bin/perl

print "Content-type:text/htmlnn";
print <<EndOfHTML;
<html><head><title>Perl Environment Variables</title></head>
<body>
<h1>Perl Environment Variables</h1>
EndOfHTML

foreach $key (sort(keys %ENV)) {
    print "$key = $ENV{$key}<br>n";
}

print "</body></html>";
```
 
保存后还要改一下权限：
```
chmod 755 /home/httpd/bugzilla.domainname.com/web/test.pl
```

再用浏览器访问一下这个页面是否正常运行了。

没问题后fcgi-perl环境就算配置好了，下面安装bugzilla：
```bash
cd /home/httpd/bugzilla.domainname.com
wget http://ftp.mozilla.org/pub/mozilla.org/webtools/bugzilla-4.0.3.tar.gz

tar -xvzf bugzilla-4.0.3.tar.gz

mv bugzilla-4.0.3 web

cd web
```
安装前要检查bugzilla所需的模块是否都安装了：
```
./checksetup.pl –check-modules
```

如果没有安装，用下面的方式安装： 
```  
yum  –disablerepo=epel install perl-DateTime perl-DateTime-TimeZone  perl-DBI perl-Template-Toolkit perl-Template-Toolkit perl-Email-Send perl-Email-MIME perl-URI perl-List-MoreUtils perl-DBD-Pg  perl-DBD-Oracle  perl-GD  perl-Chart perl-Template-GD perl-GDTextUtil perl-GDGraph perl-MIME-tools perl-libwww-perl perl-XML-Twig perl-PatchReader perl-LDAP perl-Authen-SASL perl-RadiusPerl perl-SOAP-Lite JSON-RPC perl-JSON-XS  perl-Test-Taint perl-HTML-Parser perl-HTML-Scrubber perl-Email-MIME-Attachment-Stripper  perl-Email-Reply perl-TheSchwartz perl-Daemon-Generic Math-Random-Secure
```

再次用`./checksetup.pl –check-modules`检查，发现还有几个包没安装，提示用下面命令安装，但我执行后一直安装不上：
```
/usr/bin/perl install-module.pl JSON::RPC
/usr/bin/perl install-module.pl Apache2::SizeLimit
/usr/bin/perl install-module.pl Math::Random::Secure
/usr/bin/perl install-module.pl Email::MIME
```

那就不用bugzilla安装模块了，自己安装：
```bash
perl -MCPAN -e "install Email::MIME"
perl -MCPAN -e shell
cpan>force install JSON::RPC
cpan>force install Math::Random::Secure
```
最后再检查下，应该没问题了。

编辑配置文件，把目录，数据库ip,库名，用户和密码之类的填好。
```
vim localconfig
```

然后就可以正式安装了：
```
./checksetup.pl
```
会提示输入一个邮箱和密码，这个是默认的管理员用户和密码。

安装好后就浏览器打开看看是否正常就行了。

另外数据库还需要优化一下，加入下面两行：
```
[mysqld]

max_allowed_packet=4M
ft_min_word_len=2
```
更改后要重启数据库。

下面是安装繁体中文包：
```
http://code.google.com/p/bugzilla-tw/

wget http://bugzilla-tw.googlecode.com/files/bugzilla.zh-TW.4.0.3.20120108.tar.gz
tar -xvzf bugzilla.zh-TW.4.0.3.20120108.tar.gz

cp -rf bugzilla-tw/template/zh-TW  /home/httpd/bugzilla.domainname.com/web/template
```

再打开浏览器时就会显示繁体中文了。

最后加上几个cron,bugzilla用的：
```
crontab -e

5  0 * * * cd /home/httpd/bugzilla.domainname.com/web && ./collectstats.pl
55 0 * * * cd /home/httpd/bugzilla.domainname.com/web && ./whineatnews.pl
*/15 * * * * cd /home/httpd/bugzilla.domainname.com/web && ./whine.pl
```
 

参考：
http://library.linode.com/web-servers/nginx/perl-fastcgi/centos-5

完。

