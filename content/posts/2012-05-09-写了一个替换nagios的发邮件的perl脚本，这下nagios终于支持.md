---
title: 写了一个替换nagios的发邮件的perl脚本，这下nagios终于支持发中文邮件了
author: 阿辉
date: 2012-05-09T18:03:00+00:00
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

之前nagios发邮件对中文支持有问题，最近花了点写了一个替换nagios的发邮件的perl脚本，这下nagios终于支持发中文邮件了
perl脚本如下：
<!--more-->
```perl
#!/usr/bin/perl -w
#
######################################################################################################
#requirement:
#rpm -ivh http://dag.wieers.com/rpm/packages/rpmforge-release/rpmforge-release-0.3.6-1.el5.rf.i386.rpm
#yum -y install perl-Authen-SASL perl-MIME-Base64 perl-Net-SMTP_auth perl-Net-SMTP-TLS perl-Net-SMTP-SSL
#
#脚本功能: 邮件发送脚本，支持中文
#
#作    者: Luo Hui(farmer.luo at gmail.com)
#最后时间: 2010.05.09
#版    本：Ver 0.4
#
#
#######################################################################################################

use strict;
use warnings;
use DBI();
use File::Basename;
use MIME::Base64;
use Net::SMTP;
use Net::SMTP::TLS;
use vars qw($opt_t); # 邮件主题
use vars qw($opt_m); # 发送的邮件地址
use vars qw($opt_c); # 邮件内容
use vars qw($opt_b); # 邮件编码  
use vars qw($opt_h); # 帮助
use Getopt::Std;

#smtp服务器
my $smtp_server = 'smtp.gmail.com';
#smtp服务器类型
my $smtp_type = 'SMTP_TLS';
#smtp服务器端口
my $smtp_port = 587;
#smtp认证用户名
my $username = 'test@test.com';
#smtp认证密码
my $password = 'test';
#smtp发件人
my $from_mail = 'test@test.com';

$opt_t = "";
$opt_c = "";
$opt_m = "";
$opt_b = "UTF-8";

getopt('htcmb');

if ( $opt_t eq "" || $opt_c eq "" ) {
    print "Usage:n" , basename( $0 ) , " -b 'UTF-8 | GB2312'  -m 'email address' -t 'mail title' -c 'mail content' n";
    exit(1);
}

#$opt_c = encode_base64( $opt_c );
#print $opt_c;
$opt_c =~ s/n/
/g;
$opt_c =~ s/ / /g;
#print $opt_c;

#邮件主题
my $subject = encode_base64( $opt_t, "" );

if ( $smtp_type eq 'SMTP_TLS' ) {

    my $smtp = new Net::SMTP::TLS(
        $smtp_server,
        Hello   =>      $smtp_server,
        Port    =>      $smtp_port,
        User    =>      $username,
        Password=>      $password);
    $smtp->mail($from_mail);
    $smtp->to($opt_m);
    $smtp->data;

    # send mail head
    $smtp -> datasend( "From: $from_mailn" );
    $smtp -> datasend( "To: $opt_mn" );
    #$smtp -> datasend( "MIME-Version: 1.0 n" );
    $smtp -> datasend( "Content-Type: text/html; charset=${opt_b}; n" );
    #$smtp -> datasend( "Content-Transfer-Encoding: base64n" );
    $smtp -> datasend( "Subject: =?${opt_b}?B?$subject?=nn");

    # send mail body
    $smtp -> datasend( $opt_c );
    $smtp -> dataend();

    #print "send mail to:$opt_mn";

    $smtp -> quit();
} elsif ( $smtp_type eq 'SMTP' ) {

    my $smtp = Net::SMTP -> new( Host => $smtp_server,
    #            Debug => 1,
                Hello => $smtp_server,
                ) || die "Can't connect $smtp_server $!n";

    $smtp -> auth( $username, $password ) || die "Can't authenticate: $!n";

    $smtp -> mail( $from_mail );
    $smtp -> to( $opt_m );
    $smtp -> data( );

    # send mail head
    $smtp -> datasend( "From: $from_mailn" );
    $smtp -> datasend( "To: $opt_mn" );
    #$smtp -> datasend( "MIME-Version: 1.0 n" );
    $smtp -> datasend( "Content-Type: text/html; charset=${opt_b}; n" );
    #$smtp -> datasend( "Content-Transfer-Encoding: base64n" );
    $smtp -> datasend( "Subject: =?${opt_b}?B?$subject?=nn");

    # send mail body
    $smtp -> datasend( $opt_c );
    $smtp -> dataend();

    #print "send mail to:$opt_mn";

    $smtp -> quit();

} else {
    #print "smtp type error!n";
    exit(2);
}
```