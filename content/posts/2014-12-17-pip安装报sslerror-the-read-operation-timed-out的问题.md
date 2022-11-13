---
title: 'pip安装报SSLError: The read operation timed out的问题'
author: 阿辉
date: 2014-12-17T05:40:20+00:00
categories:
- Python
tags:
- Python
keywords:
- Python
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
在windows 上安装python的模块有时报错：

```
C:Usersluohui>pip install zope.interface
Downloading/unpacking zope.interface
Cleaning up...
Exception:
Traceback (most recent call last):
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipbasecommand.py", l
ine 122, in main
status = self.run(options, args)
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipcommandsinstall.p
y", line 278, in run
requirement_set.prepare_files(finder, force_root_egg_info=self.bundle, bundl
e=self.bundle)
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipreq.py", line 1197
, in prepare_files
do_download,
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipreq.py", line 1375
, in unpack_url
self.session,
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipdownload.py", line
572, in unpack_http_url
download_hash = _download_url(resp, link, temp_location)
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipdownload.py", line
433, in _download_url
for chunk in resp_read(4096):
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpipdownload.py", line
421, in resp_read
chunk_size, decode_content=False):
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpip_vendorrequestsp
ackagesurllib3response.py", line 240, in stream
data = self.read(amt=amt, decode_content=decode_content)
File "C:Python27libsite-packagespip-1.5.6-py2.7.eggpip_vendorrequestsp
ackagesurllib3response.py", line 187, in read
data = self._fp.read(amt)
File "C:Python27libhttplib.py", line 567, in read
s = self.fp.read(amt)
File "C:Python27libsocket.py", line 380, in read
data = self._sock.recv(left)
File "C:Python27libssl.py", line 246, in recv
return self.read(buflen)
File "C:Python27libssl.py", line 165, in read
return self._sslobj.read(len)
SSLError: The read operation timed out

Storing debug log for failure in C:Usersluohuipippip.log
```
<!--more-->

解决办法，增加`--default-timeout=100`，加大超时时间。如：

```
C:Usersluohui>pip --default-timeout=100  install zope.interface
Downloading/unpacking zope.interface
Running setup.py (path:c:usersluohuiappdatalocaltemppip_build_luohuizop
e.interfacesetup.py) egg_info for package zope.interface

warning: no previously-included files matching '*.dll' found anywhere in dis
tribution
warning: no previously-included files matching '*.pyc' found anywhere in dis
tribution
warning: no previously-included files matching '*.pyo' found anywhere in dis
tribution
warning: no previously-included files matching '*.so' found anywhere in dist
ribution
Requirement already satisfied (use --upgrade to upgrade): setuptools in c:pytho
n27libsite-packagessetuptools-7.0-py2.7.egg (from zope.interface)
Installing collected packages: zope.interface
Running setup.py install for zope.interface

warning: no previously-included files matching '*.dll' found anywhere in dis
tribution
warning: no previously-included files matching '*.pyc' found anywhere in dis
tribution
warning: no previously-included files matching '*.pyo' found anywhere in dis
tribution
warning: no previously-included files matching '*.so' found anywhere in dist
ribution
building 'zope.interface._zope_interface_coptimizations' extension
C:UsersluohuiAppDataLocalProgramsCommonMicrosoftVisual C++ for Pytho
n9.0VCBinamd64cl.exe /c /nologo /Ox /MD /W3 /GS- /DNDEBUG -IC:Python27inc
lude -IC:Python27PC /Tcsrczopeinterface_zope_interface_coptimizations.c /Fo
buildtemp.win-amd64-2.7Releasesrczopeinterface_zope_interface_coptimizatio
ns.obj
_zope_interface_coptimizations.c
srczopeinterface_zope_interface_coptimizations.c(583) : warning C4244: '=
' : conversion from 'Py_ssize_t' to 'int', possible loss of data
srczopeinterface_zope_interface_coptimizations.c(1318) : warning C4244: '
=' : conversion from 'Py_ssize_t' to 'int', possible loss of data
C:UsersluohuiAppDataLocalProgramsCommonMicrosoftVisual C++ for Pytho
n9.0VCBinamd64link.exe /DLL /nologo /INCREMENTAL:NO /LIBPATH:C:Python27li
bs /LIBPATH:C:Python27PCbuildamd64 /EXPORT:init_zope_interface_coptimizations
buildtemp.win-amd64-2.7Releasesrczopeinterface_zope_interface_coptimizati
ons.obj /OUT:buildlib.win-amd64-2.7zopeinterface_zope_interface_coptimizatio
ns.pyd /IMPLIB:buildtemp.win-amd64-2.7Releasesrczopeinterface_zope_interfa
ce_coptimizations.lib /MANIFESTFILE:buildtemp.win-amd64-2.7Releasesrczopein
terface_zope_interface_coptimizations.pyd.manifest
_zope_interface_coptimizations.obj : warning LNK4197: export 'init_zope_inte
rface_coptimizations' specified multiple times; using first specification
Creating library buildtemp.win-amd64-2.7Releasesrczopeinterface_zop
e_interface_coptimizations.lib and object buildtemp.win-amd64-2.7Releasesrcz
opeinterface_zope_interface_coptimizations.exp
Skipping installation of C:Python27Libsite-packageszope__init__.py (nam
espace package)
Installing C:Python27Libsite-packageszope.interface-4.1.1-py2.7-nspkg.pt
h
Successfully installed zope.interface
Cleaning up...

 

windows pip install twisted SSLError: The read operation timed out

```