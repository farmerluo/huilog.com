---
title: MacOSX 10.8 Mountain Lion 对ntfs格式读写
author: 阿辉
date: 2012-10-10T02:49:47+00:00
categories:
- MacOSX
tags:
- MacOSX
keywords:
- MacOSX
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
MacOSX默认只支持ntfs系统的只读，但其实从mac OSX 10.6以后，系统内部是支持对ntfs读写的。可能是版权等原因没有开放。

利用系统自带方式读写NTFS系统的方法：

1. 打开终端

2. `mkdir /Volumes/DATA/`

3. `sudo mount -t ntfs -o rw,nobrowse /dev/disk1s2 /Volumes/DATA/`

如果想查看硬盘分区，可以使用diskutil list命令。

<!--more-->

注意：

1. 必须带nobrowse选项，否则使用rw参加挂上后还是只读。

2. 因为使用了nobrowse挂载，finder内将看不到这个硬盘，但是可以进入/Volumes/DATA访问这个硬盘，可以使用path finder。


其他办法：

1. Paragon NTFS，还是挺好用的，可以自动挂载ntfs格式的硬盘。收费软件$19.99.

2. Tuxera NTFS, 没用过，也是收费软件，$32.34.

3. NTFS-3G,已经很久未更新，现在在10.8能不能用，没有测试过。但是是免费的。参考：
   http://blog.crhan.com/2012/08/macosx-10-8-mountain-lion-%E8%87%AA%E5%8A%A8%E6%8C%82%E8%BD%BD%E8%AF%BB%E5%86%99-ntfs-%E5%88%86%E5%8C%BA/