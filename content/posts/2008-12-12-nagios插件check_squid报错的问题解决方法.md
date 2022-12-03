---
title: nagios插件check_squid报错的问题解决方法
author: 阿辉
date: 2008-12-12T13:22:00+00:00
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
http://workaround.org/squid/nagios-plugin/check_squid 的脚本用来检测squid很好用，但有时会报错：

`Parsing of undecoded UTF-8 will give garbage when decoding entities at /usr/lib/perl5/vendor_perl/5.8.5/LWP/Protocol.pm line 114.`

原因是：
HTML::HeadParser模块在使用parse()方法时，对没有编码的UTF-8会弄混，要保证在传值之前进行适当的编码。
参考：http://www.xinjiezuo.com/blog/?p=43

<!--more-->

解决方式是加入一行：

`ua->parse_head(0);`

跳过去就好了。

下面是改过后的check_squid.

```perl
#!/usr/bin/perl -w
#
# check_squid - Nagios check plugin for testing a Squid proxy
#
# Christoph Haas (email@christoph-haas.de)
# License: GPL 2
#
# V0.2
#

use LWP::UserAgent;
use HTTP::Request::Common qw(POST GET);
use HTTP::Headers;
use strict;

my (url, urluser, urlpass, proxy, proxyport,
        proxyuser, proxypass, expectstatus) = @ARGV;

unless (url && proxy && expectstatus)
{
        print “Usage: url urluser urlpass proxy proxyport proxyuser proxypass expectstatusn”;
                                print “ url       -> The URL to check on the internet (http://www.google.com)n";
                                print “ urluser   -> Username if the web site required authentication (- = none)n”;
                                print “ urlpass   -> Password if the web site required authentication (- = none)n”;
                                print “ proxy     -> Server that squid runs on (proxy.mydomain)n”;
                                print “ proxyport -> TCP port that Squid listens on (3128)n”;
                                print “ proxyuser -> Username if the web site required authentication (- = none)n”;
                                print “ proxypass -> Password if the web site required authentication (- = none)n”;
                                print “ expectstatus -> HTTP code that should be returnedn”;
                                print “                  (2 = anything that begins with 2)n”;
        exit -1;
}

urluser=’’ if urluser eq ‘-‘;
urlpass=’’ if urlpass eq ‘-‘;
proxyuser=’’ if proxyuser eq ‘-‘;
proxypass=’’ if proxypass eq ‘-‘;

my ua = new LWP::UserAgent;
ua->parse_head(0); //加入这行就好了

my h = HTTP::Headers->new();

if (proxy)
{
        ua->proxy([‘http’, ‘ftp’], “http://proxy:proxyport”);

        if (proxyuser)
        {
                h->proxy_authorization_basic(proxyuser,proxypass);
        }
}

if (urluser)
{
        h->authorization_basic(urluser, urlpass);
}

my req = HTTP::Request->new(‘GET’, url, h);

my res = ua->request(req);

if (res->status_line =~ /^expectstatus/)
{
        print “OK - Status: “.res->status_line.”n”;
        exit 0;
}
else
{
        print “WARNING - Status: “.res->status_line.” (but expected expectstatus…)n”;
        exit 1;
}
```