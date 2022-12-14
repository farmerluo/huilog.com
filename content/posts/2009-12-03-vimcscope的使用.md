---
title: vim+cscope的使用
author: 阿辉
date: 2009-12-03T10:53:00+00:00
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
1. vim的准备

首先 `vim --version | grep cscope`，看看vim是否支持cscope，如果不支持需要重新安装vim。最简单的是在`. /configure` 后加上`-enable-cscope`，当然可以在Makefile文件（./src/Makefile）中修改（把原来的注释去掉），这是最根本的：

`CONF_OPT_CSCOPE = --enable-cscope`

然后安装：`make && make install`

<!--more-->

2. 在目录下建立cscope索引文件

为了方便使用，编写了下面的脚本来更新cscope和ctags的索引文件：
```bash
#!/bin/sh

find . -name "*.h" -o -name "*.c" -o -name "*.cc" > cscope.files
cscope -bkq -i cscope.files
ctags -R
```

这个命令会生成三个文件：cscope.out, cscope.in.out, cscope.po.out。

其中cscope.out是基本的符号索引，后两个文件是使用"-q"选项生成的，可以加快cscope的索引速度。

这个脚本，首先使用find命令，查找当前目录及子目录中所有后缀名为".h", ".c"和".c"的文件，并把查找结果重定向到文件cscope.files中。

然后cscope根据cscope.files中的所有文件，生成符号索引文件。

最后一条命令使用ctags命令，生成一个tags文件，在vim中执行":help tags"命令查询它的用法。它可以和cscope一起使用。

上面所用到的命令参数，含义如下：

-R: 在生成索引文件时，搜索子目录树中的代码

-b: 只生成索引文件，不进入cscope的界面

-q: 生成cscope.in.out和cscope.po.out文件，加快cscope的索引速度

-k: 在生成索引文件时，不搜索/usr/include目录

-i: 如果保存文件列表的文件名不是cscope.files时，需要加此选项告诉cscope到哪儿去找源文件列表。可以使用“-”，表示由标准输入获得文件列表。

-I dir: 在-I选项指出的目录中查找头文件

-u: 扫描所有文件，重新生成交叉索引文件

-C: 在搜索时忽略大小写

-P path: 在以相对路径表示的文件前加上的path，这样，你不用切换到你数据库文件所在的目录也可以使用它了。

3. 在vim里读代码

在VIM中使用cscope非常简单，首先调用“cscope add”命令添加一个cscope数据库，然后就可以调用“cscope find”命令进行查找了。VIM支持8种cscope的查询功能，如下：例如，我们想在代码中查找调用work()函数的函数，我们可以输入：“:cs find c work”，回车后发现没有找到匹配的功能，可能并没有函数调用work()。我们再输入“:cs find s work”，查找这个符号出现的位置，现在vim列出了这个符号出现的所有位置。我们还可以进行字符串查找，它会双引号或单引号括起来的内容中查找。还可以输入一个正则表达式，这类似于egrep程序的功能。

s: 查找C语言符号，即查找函数名、宏、枚举值等出现的地方

g: 查找函数、宏、枚举等定义的位置，类似ctags所提供的功能

d: 查找本函数调用的函数

c: 查找调用本函数的函数

t: 查找指定的字符串

e: 查找egrep模式，相当于egrep功能，但查找速度快多了

f: 查找并打开文件，类似vim的find功能

i: 查找包含本文件的文