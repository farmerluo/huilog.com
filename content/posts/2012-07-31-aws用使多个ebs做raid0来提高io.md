---
title: AWS用使多个EBS做Raid0来提高IO
author: 阿辉
date: 2012-07-31T06:10:18+00:00
categories:
- AWS
tags:
- AWS
- EC2
keywords:
- AWS
- EC2
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
aws上创建raid0:

```bash
mdadm --create /dev/md0 --level=0 -c256 --raid-devices=8 /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm

echo 'DEVICE /dev/sdf /dev/sdg /dev/sdh /dev/sdi /dev/sdj /dev/sdk /dev/sdl /dev/sdm' > /etc/mdadm.conf

mdadm --detail --scan >> /etc/mdadm.conf

blockdev --setra 65536 /dev/md0

mkfs -t ext3 /dev/md0

mount /dev/md0 /data1/test 
```

<!--more-->

下面是速度测试，IO只提高了不到一倍，没有达到理想的状态。不过聊胜于无了。

```bash
[root@ip-10-160-206-223 data1]# bonnie++ -s 2048 -r 512 -d /data1/test/ -m raid0 -u root
Using uid:0, gid:0.
Writing with putc()...done
Writing intelligently...done
Rewriting...done
Reading with getc()...done
Reading intelligently...done
start 'em...done...done...done...
Create files in sequential order...done.
Stat files in sequential order...done.
Delete files in sequential order...done.
Create files in random order...done.
Stat files in random order...done.
Delete files in random order...done.
Version 1.03e       ------Sequential Output------ --Sequential Input- --Random-
                    -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
raid0            2G 83720  95 813061  95 1185231 100 84567  97 2828489  97  1187   0
                    ------Sequential Create------ --------Random Create--------
                    -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16 +++++ +++ +++++ +++ +++++ +++ +++++ +++ +++++ +++ +++++ +++
raid0,2G,83720,95,813061,95,1185231,100,84567,97,2828489,97,1187.5,0,16,+++++,+++,+++++,+++,+++++,+++,+++++,+++,+++++,+++,+++++,+++


[root@ip-10-160-206-223 data1]# bonnie++ -s 2048 -r 512 -d /data1/test/ -m raid0 -u root -b
Using uid:0, gid:0.
Writing with putc()...done
Writing intelligently...done
Rewriting...done
Reading with getc()...done
Reading intelligently...done
start 'em...done...done...done...
Create files in sequential order...done.
Stat files in sequential order...done.
Delete files in sequential order...done.
Create files in random order...done.
Stat files in random order...done.
Delete files in random order...done.
Version 1.03e       ------Sequential Output------ --Sequential Input- --Random-
                    -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
raid0            2G 87919  99 55024   6 56396   5 88632  99 2981975  99 +++++ +++
                    ------Sequential Create------ --------Random Create--------
                    -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16   319   0 +++++ +++   266   0   307   0 +++++ +++   269   0
raid0,2G,87919,99,55024,6,56396,5,88632,99,2981975,99,+++++,+++,16,319,0,+++++,+++,266,0,307,0,+++++,+++,269,0

[root@ip-10-160-206-223 data1]# bonnie++ -s 2048 -r 512 -d /data2/test/ -m ebs -u root -b      
Using uid:0, gid:0.
Writing with putc()...done
Writing intelligently...done
Rewriting...done
Reading with getc()...done
Reading intelligently...done
start 'em...done...done...done...
Create files in sequential order...done.
Stat files in sequential order...done.
Delete files in sequential order...done.
Create files in random order...done.
Stat files in random order...done.
Delete files in random order...done.
Version 1.03e       ------Sequential Output------ --Sequential Input- --Random-
                    -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
ebs              2G 78798  88 29578   3 36800   3 88879  99 3027079  99 +++++ +++
                    ------Sequential Create------ --------Random Create--------
                    -Create-- --Read--- -Delete-- -Create-- --Read--- -Delete--
              files  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP  /sec %CP
                 16   443   0 +++++ +++   416   0   435   0 +++++ +++   423   0
ebs,2G,78798,88,29578,3,36800,3,88879,99,3027079,99,+++++,+++,16,443,0,+++++,+++,416,0,435,0,+++++,+++,423,0
```