---
title: 用nagios监控mysql replication主备间的同步情况
author: 阿辉
date: 2011-03-30T14:53:00+00:00
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
# cd /etc/nagios/command

# wget http://www.james.rcpt.to/svn/trunk/nagios/check_mysql_replication/check_mysql_replication.pl

# chmod 755 check_mysql_replication.pl

看看用法：

<!--more-->
```bash
# ./check_mysql_replication.pl -h
check_mysql_replication.pl: check replication between MySQL database instances

 check_replication.pl [ --slave ] [ --slave-pass ]
 [ --slave-port ] [ --slave-user ] [ --master ]
 [ --master-pass ] [ --master-port ] [ --master-user ]
 [ --crit ] [ --warn ] [ --check-random-database ]
 [ --table-rows-diff-absolute-crit ]
 [ --table-rows-diff-absolute-warn ]
 [ --schema ]

 --slave          - MySQL instance running as a slave server
 --slave-port        - port for the slave
 --slave-user     - Username with File/Process/Super privs
 --slave-pass     - Password for above user
 --master         - MySQL instance running as server (override)
 --master-port       - port for the master (override)
 --master-user    - Username for master (override)
 --master-pass    - Password for master
 --crit      - Number of complete master binlogs for critical state
 --warn      - Number of complete master binlog for warning state
 --check-random-database - Select a random DB from the slave's list of
                databases and compare to the master's information for
                these (need SELECT priv)
 --table-rows-diff-absolute-crit - If we do the check-random-database,
                then ensure that the change in row count between master and
                slave is below this threshold, and go critical if not
 --table-rows-diff-absolute-warn - If we do the check-random-database,
                then ensure that the change in row count between master and
                slave is below this threshold, and go warning if not
 --schema           - The database schema to use
 --help                 - This help page
 --version              - Script version information

By default, you should use your configured replication user, as you will
then only need to specify the user and password once, and this script will
find the master from the slave's running configuration.

Critical and warning values are now measured as amount of a complete master
sized binlog. If your master has the default 1GB binlog size, then specifying
a warning value of 0.1 means that your will let the slave get 100MB out of
sync before warning; you may want to set warning to 0.01, and critical at 0.1.

MySQL 3: GRANT File, Process on *.* TO repl@192.168.0.% IDENTIFIED BY
MySQL 4: GRANT Super, Replication client on *.* TO repl@192.168.0.% IDE...

If you want to use the check-random-database option, then the user needs
SELECT privileges on all replicated tables on the master and the slave.

Note: Any mysqldump tables (for backups) may lock large tables for a long
time. If you dump from your slave for this, then your master will gallop
away from your slave, and the difference will become large. The trick is to
set 'crit' above this differnce and 'warn' below.

If you are using the host name "localhost" to connect to port forwards, you'll
probably hit the issue where MySQL uses the named pipe (socket) on the file
system instead of there TCP loopback address. Use host name "127.0.0.1".

(c) 2010 James Bromberger, james@rcpt.to, www.james.rcpt.to
```

先在命令行运行测试：
```bash
# /etc/nagios/command/check_mysql_replication.pl --slave 192.168.1.132 --slave-user user --slave-pass passwd --master 192.168.1.131 --master-user user --master-pass passwd
OK: 0.000 diff, 0 secs, 192.168.1.131:3306 (5.0.77-log) -> 192.168.1.132:3306 (5.0.77-log)
```
运行成功，我们配置到nagios上去。
```
# vim objects/commands.cfg
```
加上一段：
```
define command {
        command_name    check_mysql_replication
        command_line    /etc/nagios/command/check_mysql_replication.pl --slave $HOSTADDRESS$ --slave-user $ARG1$ --slave-pass $ARG2$ --
master $ARG3$ --master-user $ARG4$ --master-pass $ARG5$
        }
```
```
# vim objects/hosts.cfg
```
加上一段：
```
define service{
        use                             generic-service         ; Name of service template to use
        host_name                       mysql_slave
        service_description             mysql replication
        check_command                   check_mysql_replication!user!passwd!192.168.1.131!user!passwd
        notifications_enabled           1
        }
```
然后重新载入nagios配置文件：
```
# service nagios reload
```
过了五分钟才发现结果并不是我想要的，在nagios内发现检测的结果和手动运行脚本不一样，提示：

Service check did not exit properly

看到这个错误，就又回去检查了两篇配置，发现并没有什么问题。

再仔细看了下报错，是说脚本的exit不正确，立马看了下脚本，发现exit是类似这样的：
```
exit 3;

exit 2;

exit 1;

exit 0;
```
正好之前自己也写过检测redis的nagios脚本。依稀记得nagios对此是有要求的，不能直接这么用。然后把脚本做了如下更改：
```
exit 3 改成 exit $ERRORS{"UNKNOWN"};
exit 2 改成 exit $ERRORS{"CRITICAL"};
exit 1 改成 exit $ERRORS{"WARNING"};
exit 0 改成 exit $ERRORS{"OK"};
```
并在开头的use部分加上一行：

use utils qw($TIMEOUT %ERRORS &print_revision &support);

另外还需要把下面这行加在脚本的前10行内：
```
# nagios: -epn
```
意思是不用Nagios自带的嵌入式Perl解释器运行此脚本。

再等几分钟，发现脚本在nagios内运行正常了。

怪异的是我在网上看到很多人都在用这个脚本，没见有人说碰到这个问题。难道和使用的版本或是配置有关？