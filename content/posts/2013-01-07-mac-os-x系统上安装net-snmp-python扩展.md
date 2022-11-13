---
title: Mac OS X系统上安装net-snmp python扩展
author: 阿辉
date: 2013-01-07T08:50:14+00:00
categories:
- MacOSX
tags:
- MacOSX
- Python
keywords:
- MacOSX
- Python
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
发现Mac OS X系统下虽然有net-snmp,但是没有安装net-snmp python的扩展，而这个扩展用pip或easy_install也是安装不了的。只能通过下载源码安装。

下载：
```
luohui@mac:~/Downloads > wget http://downloads.sourceforge.net/project/net-snmp/net-snmp/5.6.2/net-snmp-5.6.2.tar.gz?r=http%3A%2F%2Fsourceforge.net%2Fprojects%2Fnet-snmp%2Ffiles%2Fnet-snmp%2F5.6.2%2F&ts=1357547749&use_mirror=nchc
luohui@mac:~/Downloads > tar -xvzf net-snmp-5.6.2.tar.gz
luohui@mac:~/Downloads > cd net-snmp-5.6.2
luohui@mac:~/Downloads/net-snmp-5.6.2 > cd python/
luohui@mac:~/Downloads/net-snmp-5.6.2/python > ls
LICENSE  README   netsnmp  setup.py
```
<!--more-->

编译：
```
luohui@mac:~/Downloads/net-snmp-5.6.2/python > python setup.py build
running build
running build_py
creating build
creating build/lib.macosx-10.8-intel-2.7
creating build/lib.macosx-10.8-intel-2.7/netsnmp
copying netsnmp/__init__.py -> build/lib.macosx-10.8-intel-2.7/netsnmp
copying netsnmp/client.py -> build/lib.macosx-10.8-intel-2.7/netsnmp
creating build/lib.macosx-10.8-intel-2.7/netsnmp/tests
copying netsnmp/tests/__init__.py -> build/lib.macosx-10.8-intel-2.7/netsnmp/tests
copying netsnmp/tests/test.py -> build/lib.macosx-10.8-intel-2.7/netsnmp/tests
running build_ext
building 'netsnmp.client_intf' extension
creating build/temp.macosx-10.8-intel-2.7
creating build/temp.macosx-10.8-intel-2.7/netsnmp
clang -fno-strict-aliasing -fno-common -dynamic -g -Os -pipe -fno-common -fno-strict-aliasing -fwrapv -mno-fused-madd -DENABLE_DTRACE -DMACOSX -DNDEBUG -Wall -Wstrict-prototypes -Wshorten-64-to-32 -DNDEBUG -g -Os -Wall -Wstrict-prototypes -DENABLE_DTRACE -arch i386 -arch x86_64 -pipe -I/System/Library/Frameworks/Python.framework/Versions/2.7/include/python2.7 -c netsnmp/client_intf.c -o build/temp.macosx-10.8-intel-2.7/netsnmp/client_intf.o
clang: warning: argument unused during compilation: '-mno-fused-madd'
netsnmp/client_intf.c:371:18: warning: format specifies type 'unsigned long' but the argument has type 'oid' (aka 'unsigned int')
      [-Wformat]
        sprintf(buf,".%lu",*objid++);
        ~~~~~~~~~~~~~~~~^~~~~~~~~~~~
                      %u
/usr/include/secure/_stdio.h:49:56: note: expanded from macro 'sprintf'
  __builtin___sprintf_chk (str, 0, __darwin_obsz(str), __VA_ARGS__)
                                                       ^
netsnmp/client_intf.c:389:24: warning: format specifies type 'unsigned long *' but the argument has type 'oid *'
      (aka 'unsigned int *') [-Wformat]
         sscanf(cp, "%lu", objid++);
                     ~~^   ~~~~~~~
                     %u
netsnmp/client_intf.c:399:18: warning: format specifies type 'unsigned long *' but the argument has type 'oid *'
      (aka 'unsigned int *') [-Wformat]
   sscanf(cp, "%lu", objid++);
               ~~^   ~~~~~~~
               %u
netsnmp/client_intf.c:723:20: warning: format specifies type 'unsigned long *' but the argument has type 'oid *'
      (aka 'unsigned int *') [-Wformat]
     sscanf(cp, "%lu", &(doid_arr[(*doid_arr_len)++]));
                 ~~^   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                 %u
netsnmp/client_intf.c:1036:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    verbose = py_netsnmp_attr_long(pkg, "verbose");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1474:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1475:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1673:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_num = py_netsnmp_attr_long(session, "ErrorNum");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1674:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_ind = py_netsnmp_attr_long(session, "ErrorInd");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1684:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1685:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1898:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_num = py_netsnmp_attr_long(session, "ErrorNum");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1899:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_ind = py_netsnmp_attr_long(session, "ErrorInd");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1909:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1910:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2250:17: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      err_num = py_netsnmp_attr_long(session, "ErrorNum");
              ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2251:17: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      err_ind = py_netsnmp_attr_long(session, "ErrorInd");
              ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2261:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      best_guess = py_netsnmp_attr_long(session, "BestGuess");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2262:22: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                   ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2481:17: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    use_enums = py_netsnmp_attr_long(session, "UseEnums");
              ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2483:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
21 warnings generated.
netsnmp/client_intf.c:262:8: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
        len = STRLEN(buf);
              ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:270:27: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
                    len = STRLEN(buf);
                          ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:277:21: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
              len = STRLEN(buf);
                    ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:286:18: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
           len = STRLEN(buf);
                 ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:291:23: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
           len = var->val_len;
               ~ ~~~~~^~~~~~~
netsnmp/client_intf.c:293:22: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
               len = buf_len;
                   ~ ^~~~~~~
netsnmp/client_intf.c:300:17: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
          len = STRLEN(buf);
                ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:309:17: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
          len = STRLEN(buf);
                ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:328:17: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
          len = STRLEN(buf);
                ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:340:19: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
            len = STRLEN(buf);
                  ^~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:371:18: warning: format specifies type 'unsigned long' but the argument has type 'oid' (aka 'unsigned int')
      [-Wformat]
        sprintf(buf,".%lu",*objid++);
        ~~~~~~~~~~~~~~~~^~~~~~~~~~~~
                      %u
/usr/include/secure/_stdio.h:49:56: note: expanded from macro 'sprintf'
  __builtin___sprintf_chk (str, 0, __darwin_obsz(str), __VA_ARGS__)
                                                       ^
netsnmp/client_intf.c:389:24: warning: format specifies type 'unsigned long *' but the argument has type 'oid *'
      (aka 'unsigned int *') [-Wformat]
         sscanf(cp, "%lu", objid++);
                     ~~^   ~~~~~~~
                     %u
netsnmp/client_intf.c:399:18: warning: format specifies type 'unsigned long *' but the argument has type 'oid *'
      (aka 'unsigned int *') [-Wformat]
   sscanf(cp, "%lu", objid++);
               ~~^   ~~~~~~~
               %u
netsnmp/client_intf.c:495:14: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
   int len = STRLEN(name);
             ^~~~~~~~~~~~
netsnmp/client_intf.c:46:24: note: expanded from macro 'STRLEN'
#define STRLEN(x) (x ? strlen(x) : 0)
                     ~ ^~~~~~~~~
netsnmp/client_intf.c:648:21: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
     *oid_arr_len = newname_len;
                  ~ ^~~~~~~~~~~
netsnmp/client_intf.c:669:22: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
      *oid_arr_len = newname_len;
                   ~ ^~~~~~~~~~~
netsnmp/client_intf.c:681:22: warning: implicit conversion loses integer precision: 'u_long' (aka 'unsigned long') to 'oid'
      (aka 'unsigned int') [-Wshorten-64-to-32]
           *op = tp->subid;
               ~ ~~~~^~~~~
netsnmp/client_intf.c:686:47: warning: implicit conversion loses integer precision: 'long' to 'int' [-Wshorten-64-to-32]
         *oid_arr_len = newname + MAX_OID_LEN - op;
                      ~ ~~~~~~~~~~~~~~~~~~~~~~^~~~
netsnmp/client_intf.c:723:20: warning: format specifies type 'unsigned long *' but the argument has type 'oid *'
      (aka 'unsigned int *') [-Wformat]
     sscanf(cp, "%lu", &(doid_arr[(*doid_arr_len)++]));
                 ~~^   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                 %u
netsnmp/client_intf.c:911:67: warning: implicit conversion loses integer precision: 'long' to 'int' [-Wshorten-64-to-32]
               if (retry_nosuch && (pdu = snmp_fix_pdu(*response, command))) {
                                          ~~~~~~~~~~~~            ^~~~~~~
netsnmp/client_intf.c:937:68: warning: implicit conversion loses integer precision: 'long' to 'int' [-Wshorten-64-to-32]
               strlcpy(err_str, (char*)snmp_errstring((*response)->errstat),
                                       ~~~~~~~~~~~~~~ ~~~~~~~~~~~~~^~~~~~~
netsnmp/client_intf.c:940:33: warning: implicit conversion loses integer precision: 'long' to 'int' [-Wshorten-64-to-32]
               *err_ind = (*response)->errindex;
                        ~ ~~~~~~~~~~~~~^~~~~~~~
netsnmp/client_intf.c:941:38: warning: implicit conversion loses integer precision: 'long' to 'int' [-Wshorten-64-to-32]
               status = (*response)->errstat;
                      ~ ~~~~~~~~~~~~~^~~~~~~
netsnmp/client_intf.c:1036:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    verbose = py_netsnmp_attr_long(pkg, "verbose");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1246:17: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'u_int'
      (aka 'unsigned int') [-Wshorten-64-to-32]
                      session.securityAuthProtoLen,
                      ~~~~~~~~^~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1282:15: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'u_int'
      (aka 'unsigned int') [-Wshorten-64-to-32]
                    session.securityAuthProtoLen,
                    ~~~~~~~~^~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1474:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1475:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1673:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_num = py_netsnmp_attr_long(session, "ErrorNum");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1674:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_ind = py_netsnmp_attr_long(session, "ErrorInd");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1684:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1685:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1898:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_num = py_netsnmp_attr_long(session, "ErrorNum");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1899:15: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    err_ind = py_netsnmp_attr_long(session, "ErrorInd");
            ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1909:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:1910:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2040:55: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
        oid_arr_broken_check_len[varlist_ind] = vars->name_length;
                                              ~ ~~~~~~^~~~~~~~~~~
netsnmp/client_intf.c:2155:61: warning: implicit conversion loses integer precision: 'size_t' (aka 'unsigned long') to 'int'
      [-Wshorten-64-to-32]
              oid_arr_broken_check_len[varlist_ind] = vars->name_length;
                                                    ~ ~~~~~~^~~~~~~~~~~
netsnmp/client_intf.c:2250:17: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      err_num = py_netsnmp_attr_long(session, "ErrorNum");
              ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2251:17: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      err_ind = py_netsnmp_attr_long(session, "ErrorInd");
              ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2261:20: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      best_guess = py_netsnmp_attr_long(session, "BestGuess");
                 ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2262:22: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
      retry_nosuch = py_netsnmp_attr_long(session, "RetryNoSuch");
                   ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2481:17: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    use_enums = py_netsnmp_attr_long(session, "UseEnums");
              ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
netsnmp/client_intf.c:2483:18: warning: implicit conversion loses integer precision: 'long long' to 'int' [-Wshorten-64-to-32]
    best_guess = py_netsnmp_attr_long(session, "BestGuess");
               ~ ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
44 warnings generated.
clang -bundle -undefined dynamic_lookup -Wl,-F. -arch i386 -arch x86_64 build/temp.macosx-10.8-intel-2.7/netsnmp/client_intf.o -lnetsnmp -lcrypto -o build/lib.macosx-10.8-intel-2.7/netsnmp/client_intf.so
```
有很多警告，不用管它。

测试：
```
luohui@mac:~/Downloads/net-snmp-5.6.2/python > python setup.py test
running test
running egg_info
creating netsnmp_python.egg-info
writing netsnmp_python.egg-info/PKG-INFO
writing top-level names to netsnmp_python.egg-info/top_level.txt
writing dependency_links to netsnmp_python.egg-info/dependency_links.txt
writing manifest file 'netsnmp_python.egg-info/SOURCES.txt'
reading manifest file 'netsnmp_python.egg-info/SOURCES.txt'
writing manifest file 'netsnmp_python.egg-info/SOURCES.txt'
running build_ext
copying build/lib.macosx-10.8-intel-2.7/netsnmp/client_intf.so -> netsnmp
testFuncs (netsnmp.tests.test.BasicTests) ...
---v1 GET tests -------------------------------------

v1 snmpget result:  (None,)

v1 get var:  .1.3.6.1.2.1.1.1 0 = None ( None )
---v1 GETNEXT tests-------------------------------------

v1 snmpgetnext result:  (None,)

v1 getnext var:  .1.3.6.1.2.1.1.1 0 = None ( None )
---v1 SET tests-------------------------------------

v1 snmpset result:  0

v1 set var:  sysLocation 0 = my new location ( None )
---v1 walk tests-------------------------------------

v1 varlist walk in:
   system  = None ( None )
v1 snmpwalk result:  ()

---v1 walk 2-------------------------------------

v1 varbind walk in:
v1 snmpwalk result (should be = orig):  ()

system  = None ( None )
---v1 multi-varbind test-------------------------------------

v1 sess.get result:  (None, None, None)

sysUpTime 0 = None ( None )
sysContact 0 = None ( None )
sysLocation 0 = None ( None )
v1 sess.getnext result:  (None, None, None)

sysUpTime 0 = None ( None )
sysContact 0 = None ( None )
sysLocation 0 = None ( None )
v1 sess.getbulk result:  None

sysUpTime  = None ( None )
sysORLastChange  = None ( None )
sysORID  = None ( None )
sysORDescr  = None ( None )
sysORUpTime  = None ( None )
---v1 set2-------------------------------------

v1 sess.set result:  0

---v1 walk3-------------------------------------

v1 sess.walk result:  ()

---v2c get-------------------------------------

^C
interrupted
luohui@mac:~/Downloads/net-snmp-5.6.2/python >
```

像上面这些输出基本就没问题了。
安装：
```
luohui@mac:~/Downloads/net-snmp-5.6.2/python > python setup.py install
running install
error: can't create or remove files in install directory

The following error occurred while trying to add or remove files in the
installation directory:

    [Errno 13] Permission denied: '/Library/Python/2.7/site-packages/test-easy-install-8562.write-test'

The installation directory you specified (via --install-dir, --prefix, or
the distutils default setting) was:

    /Library/Python/2.7/site-packages/

Perhaps your account does not have write access to this directory?  If the
installation directory is a system-owned directory, you may need to sign in
as the administrator or "root" account.  If you do not have administrative
access to this machine, you may wish to choose a different installation
directory, preferably one that is listed in your PYTHONPATH environment
variable.

For information on other options, you may wish to consult the
documentation at:

  http://peak.telecommunity.com/EasyInstall.html

Please make the appropriate changes for your system and try again.
```

报错，需要用sudo权限安装：
```
luohui@mac:~/Downloads/net-snmp-5.6.2/python > sudo python setup.py install
Password:
running install
running bdist_egg
running egg_info
writing netsnmp_python.egg-info/PKG-INFO
writing top-level names to netsnmp_python.egg-info/top_level.txt
writing dependency_links to netsnmp_python.egg-info/dependency_links.txt
reading manifest file 'netsnmp_python.egg-info/SOURCES.txt'
writing manifest file 'netsnmp_python.egg-info/SOURCES.txt'
installing library code to build/bdist.macosx-10.8-intel/egg
running install_lib
running build_py
running build_ext
creating build/bdist.macosx-10.8-intel
creating build/bdist.macosx-10.8-intel/egg
creating build/bdist.macosx-10.8-intel/egg/netsnmp
copying build/lib.macosx-10.8-intel-2.7/netsnmp/__init__.py -> build/bdist.macosx-10.8-intel/egg/netsnmp
copying build/lib.macosx-10.8-intel-2.7/netsnmp/client.py -> build/bdist.macosx-10.8-intel/egg/netsnmp
copying build/lib.macosx-10.8-intel-2.7/netsnmp/client_intf.so -> build/bdist.macosx-10.8-intel/egg/netsnmp
creating build/bdist.macosx-10.8-intel/egg/netsnmp/tests
copying build/lib.macosx-10.8-intel-2.7/netsnmp/tests/__init__.py -> build/bdist.macosx-10.8-intel/egg/netsnmp/tests
copying build/lib.macosx-10.8-intel-2.7/netsnmp/tests/test.py -> build/bdist.macosx-10.8-intel/egg/netsnmp/tests
byte-compiling build/bdist.macosx-10.8-intel/egg/netsnmp/__init__.py to __init__.pyc
byte-compiling build/bdist.macosx-10.8-intel/egg/netsnmp/client.py to client.pyc
byte-compiling build/bdist.macosx-10.8-intel/egg/netsnmp/tests/__init__.py to __init__.pyc
byte-compiling build/bdist.macosx-10.8-intel/egg/netsnmp/tests/test.py to test.pyc
creating stub loader for netsnmp/client_intf.so
byte-compiling build/bdist.macosx-10.8-intel/egg/netsnmp/client_intf.py to client_intf.pyc
creating build/bdist.macosx-10.8-intel/egg/EGG-INFO
copying netsnmp_python.egg-info/PKG-INFO -> build/bdist.macosx-10.8-intel/egg/EGG-INFO
copying netsnmp_python.egg-info/SOURCES.txt -> build/bdist.macosx-10.8-intel/egg/EGG-INFO
copying netsnmp_python.egg-info/dependency_links.txt -> build/bdist.macosx-10.8-intel/egg/EGG-INFO
copying netsnmp_python.egg-info/top_level.txt -> build/bdist.macosx-10.8-intel/egg/EGG-INFO
writing build/bdist.macosx-10.8-intel/egg/EGG-INFO/native_libs.txt
zip_safe flag not set; analyzing archive contents...
creating dist
creating 'dist/netsnmp_python-1.0a1-py2.7-macosx-10.8-intel.egg' and adding 'build/bdist.macosx-10.8-intel/egg' to it
removing 'build/bdist.macosx-10.8-intel/egg' (and everything under it)
Processing netsnmp_python-1.0a1-py2.7-macosx-10.8-intel.egg
Copying netsnmp_python-1.0a1-py2.7-macosx-10.8-intel.egg to /Library/Python/2.7/site-packages
Adding netsnmp-python 1.0a1 to easy-install.pth file

Installed /Library/Python/2.7/site-packages/netsnmp_python-1.0a1-py2.7-macosx-10.8-intel.egg
Processing dependencies for netsnmp-python==1.0a1
Finished processing dependencies for netsnmp-python==1.0a1
luohui@mac:~/Downloads/net-snmp-5.6.2/python >
```

安装好了，进入ipython,import net-snmp不报错说明就没问题了。