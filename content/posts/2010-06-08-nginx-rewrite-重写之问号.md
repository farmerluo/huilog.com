---
title: nginx rewrite 重写之问号
author: 阿辉
date: 2010-06-08T18:05:00+00:00
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
朋友让我帮写一个nginx rewrite，大致如下：

把用户访问的URL：

http://viptest.test.com/media_files/read_pic_example.html?200904/23/40046803_1240465896n5RK.jpg

转到真实的URL：

http://viptest.test.com/attachments/200904/23/40046803_1240465896n5RK.jpg

一看，很简单，很快就写出来了。如下：
```
rewrite  ^(.)media_files/read_pic_example.html?(.) 1attachments/2  break;
```
结果他一试不行，404错误，说明跟本就没有rewrite。
<!--more-->

仔细检查，正则表达式并没有问题，应该是nginx处理的时候有些比较特殊的地方。
```
media_files/read_pic_example.html?(._)
```
这里面只有.和?比较特殊，感觉像是?的问题，所以立马去掉问号试了下：
```
rewrite  ^(._)media_files/read_pic_example.html(.) 1attachments/2  break;
```
然后访问：http://viptest.test.com/media_files/read_pic_example.html200904/23/40046803_1240465896n5RK.jpg

可以看到图片。

从这个测试来分析，?后的内容好像不参与正则匹配了。查了查文档，?后的内容其实有一个内置变量，就是query_string，更改了下rewrite：
```
rewrite  ^(._)media_files/read_pic_example.html 1attachments/$query_string  break;
```
再访问：

http://viptest.test.com/media_files/read_pic_example.html?200904/23/40046803_1240465896n5RK.jpg

这下OK了，搞掂。