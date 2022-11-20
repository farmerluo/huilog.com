---
title: 自己写了一个perl脚本检测redis(nagios插件)
author: 阿辉
date: 2010-05-12T16:45:00+00:00
categories:
- Nagios
tags:
- Nagios
keywords:
- Nagios
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
放在我的code库内了，http://farmerluo.googlecode.com/files/check_redis.pl

介绍下怎么安装：

脚本用到了perl的Redis库，需要先安装这个：
```bash
# perl -MCPAN -e shell
# install Redis
wget http://farmerluo.googlecode.com/files/check_redis.pl
cp check_redis.pl  /etc/nagios/command/
chown cacti.nagios check_redis.pl
```
<!--more-->

在nagios内加入这个插件：
```bash
vi /etc/nagios/objects/command.cfg

# ‘check_redis’ command definition
define command{
command_name    check_redis
command_line    /etc/nagios/command/check_redis.pl -h HOSTADDRESS ARG1
}
```

加入一个服务：
```bash
vi /etc/nagios/objects/linuxhost.cfg

define service{
use                             generic-service         ; Name of service template to use
host_name                       memcached.ha2,memcached.web2
service_description             redis
check_command                   check_redis
notifications_enabled           1
}
```

检查下nagios配置是否正解：
```bash
nagios -v  /etc/nagios/nagios.cfg

Nagios Core 3.2.1
Copyright (c) 2009-2010 Nagios Core Development Team and Community Contributors
Copyright (c) 1999-2009 Ethan Galstad
Last Modified: 03-09-2010
License: GPL

Website: http://www.nagios.org
Reading configuration data…
Read main config file okay…
Processing object config file ‘/etc/nagios/objects/commands.cfg’…
Processing object config file ‘/etc/nagios/objects/contacts.cfg’…
Processing object config file ‘/etc/nagios/objects/timeperiods.cfg’…
Processing object config file ‘/etc/nagios/objects/templates.cfg’…
Processing object config file ‘/etc/nagios/objects/linuxhosts.cfg’…
Processing object config file ‘/etc/nagios/objects/windows.cfg’…
Read object config files okay…

Running pre-flight check on configuration data…

Checking services…
Checked 32 services.
Checking hosts…
Checked 14 hosts.
Checking host groups…
Checked 4 host groups.
Checking service groups…
Checked 0 service groups.
Checking contacts…
Checked 3 contacts.
Checking contact groups…
Checked 2 contact groups.
Checking service escalations…
Checked 0 service escalations.
Checking service dependencies…
Checked 0 service dependencies.
Checking host escalations…
Checked 0 host escalations.
Checking host dependencies…
Checked 0 host dependencies.
Checking commands…
Checked 29 commands.
Checking time periods…
Checked 5 time periods.
Checking for circular paths between hosts…
Checking for circular host and service dependencies…
Checking global event handlers…
Checking obsessive compulsive processor commands…
Checking misc settings…

Total Warnings: 0
Total Errors:   0

Things look okay - No serious problems were detected during the pre-flight check
```
没问题，我们重新载入配置。
```bash
service nagios reload
```
再介绍一下用perl写nagios插件需要注意的地方：

总是要生成一些输出内容；
加上引用’use utils’并引用些通用模块来输出(TIMEOUT %ERRORS &print_revision &支持等)；
总 是知道一些Perl插件的标准习惯，如：
退出时总是exit带 着ERRORS{CRITICAL}、ERRORS{OK}等；
使用getopt函数来处理命令行；
程序处理超 时问题；
当没有命令参数时要给出可调用print_usage；
使用标准的命令行选项开关(象-H ‘host’、-V ‘version’等)。

check_redisl.pl代码：
```perl
#!/usr/bin/perl

# nagios: -epn

################################################################################
# check_redis - Nagios Plugin for Redis checks.
#
# @author  farmer.luo at gmail.com
# @date    2012-01-15
# @license GPL v2
#
# check_nagios.pl -h -p -w -c
#
# Run the script need:
#
# perl -MCPAN -e shell
# install Redis
#
################################################################################

use strict;
use warnings;
use Redis;
use File::Basename;
use utils qw(TIMEOUT %ERRORS &print_revision &support);
use Time::Local;
use vars qw(opt_h); # Redis
use vars qw(opt_p); # Redis
use vars qw(opt_w); # ʱuse vars qw(opt_c); # ʱuse Getopt::Std;

opt_h = “”;
opt_p = “6379”;
opt_w = 5;
opt_c = 10;
my r = “”;
my role = “master”;

getopt(‘hpwcd’);

if ( opt_h eq “” ) {
        help();
        exit(1);
}

my start = time();

redis_connect();

# print @;
if ( @ ) {
        print “UNKNOWN - cann’t connect to redis server:” . opt_h . “.”;
        exit ERRORS{“UNKNOWN”};
}



#sleep(3);
my stop = time();

my run = stop - start;

if ( run > opt_c ) {

        print “CRITICAL - redis server(“ . opt_h . “) run for “ . run . “ seconds!”;
        exit ERRORS{“CRITICAL”};

} elsif ( run > opt_w ) {

        print “WARNING - redis server(“ . opt_h . “) run for “ . run . “ seconds!”;
        exit ERRORS{“WARNING”};

} else {

        redis_info();

#       print “role = “ . role;

        if ( role eq “master” ){

                if ( redis_set() ) {
                       print “WARNING - redis server:” . opt_h . “,set key error.”;
                       exit ERRORS{“WARNING”};
                }

                if ( redis_get() ) {
                       print “WARNING - redis server:” . opt_h . “,get key error.”;
                       exit ERRORS{“WARNING”};
                }

                if ( redis_del() ) {
                       print “WARNING - redis server:” . opt_h . “,del key error.”;
                       exit ERRORS{“WARNING”};
                }

        }

        redis_quit();
        exit ERRORS{“OK”};

}


sub help{

        die “Usage:n” , basename( 0 ) ,  “ -h hostname -p port -w warning time -c critical time -d down timen”

}

sub redis_connect{

        my redis_hp = opt_h . “:” . opt_p;

        eval{ r = Redis->new( server => redis_hp ); };

}

sub redis_set{

        r->set( redis_nagios_key => ‘test’ ) || return 1;

        return 0;
}

sub redis_get{

        my value = r->get( ‘redis_nagios_key’ ) || return 1;

        return 0;
}

sub redis_del{

        r->del( ‘redis_nagios_key’ ) || return 1;

        return 0;
}

sub redis_info{

        my info_hash = r->info;

        print “OK - redis server(“ . opt_h . “) info:”;

        while ( my (key, value) = each(%info_hash) ) {
            print “key => value, “;
        }

        my %info = %info_hash;

        role = info{“role”};
}

sub redis_quit{

        $r->quit();

}
```

参考：
http://nagios-cn.sourceforge.net/nagios-cn/develope.html