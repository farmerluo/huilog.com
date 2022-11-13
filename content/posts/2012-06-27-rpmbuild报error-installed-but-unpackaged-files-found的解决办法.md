---
title: 'rpmbuild报error: Installed (but unpackaged) file(s) found的解决办法'
author: 阿辉
date: 2012-06-27T14:35:00+00:00
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
我在打包时出错：
```
Processing files: php-debuginfo-5.3.10-1.x86_64
Checking for unpackaged file(s): /usr/lib/rpm/check-files /root/rpmbuild/BUILDROOT/php-5.3.10-1.x86_64
error: Installed (but unpackaged) file(s) found:
   /.channels/.alias/pear.txt
   /.channels/.alias/pecl.txt
   /.channels/.alias/phpdocs.txt
   /.channels/__uri.reg
   /.channels/doc.php.net.reg
   /.channels/pear.php.net.reg
   /.channels/pecl.php.net.reg
   /.depdb
   /.depdblock
   /.filemap
   /.lock


RPM build errors:
    Installed (but unpackaged) file(s) found:
   /.channels/.alias/pear.txt
   /.channels/.alias/pecl.txt
   /.channels/.alias/phpdocs.txt
   /.channels/__uri.reg
   /.channels/doc.php.net.reg
   /.channels/pear.php.net.reg
   /.channels/pecl.php.net.reg
   /.depdb
   /.depdblock
   /.filemap
   /.lock
```

<!--more-->

网上查询，解决办法有：

1. 在`/usr/lib/rpm/macros`文件中有一个定义:`%_unpackaged_files_terminate_build 1`,把1改为0只警告

2. `make install`后删除这些文件：
```
rm -rf %{buildroot}
make INSTALL_ROOT=%{buildroot} install

rm -rf %{buildroot}/.channels/.alias/pear.txt %{buildroot}/.channels/.alias/pecl.txt %{buildroot}/.channels/__uri.reg %{buildroot}/.channels/pear.php.net.reg %{buildroot}/.channels/pecl.php.net.reg %{buildroot}/.depdb %{buildroot}/.depdblock %{buildroot}/.filemap %{buildroot}/.lock 
```

3. 把这些文件加进去
```
%files

%dir %{_prefix}/.channels

%dir %{_prefix}/.channels/.alias/

%{_prefix}/.channels/.alias/pear.txt
```
