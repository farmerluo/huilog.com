---
title: svn自动更新脚本svnsshup.exp
author: 阿辉
date: 2010-11-26T16:31:00+00:00
categories:
- SVN
tags:
- SVN
keywords:
- SVN
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
其中`send -- "/usr/bin/svn upr"`改成`send -- "/usr/bin/nohup /usr/bin/svn up & r"`会更好，前者在内网没问题，在公网可能会不行，可能会因为执行时间过长而中断。

这个脚本还需要一个pwd.txt文件，pwd.txt文件用于存放需要更新的服务器列表信息，格式为：
```
ip1     username       passwd

ip2     username       passwd
...
```
<!--more-->
因为我目前用的是普通用户，如果这里是用root,那脚本里的`expect "$"`需要改成`expect "#"`

svnsshup.exp为：
```bash
#!/usr/bin/expect -d


set timeout 90
#exp_internal 1
proc do_ssh_login {host username pass} {
    set timeout_case 0
    set done 1

    send_user "n"
    spawn ssh $username@$host
    send_user "Connecting host $hostn"
    while {$done} {

        expect {
            timeout {
                switch -- $timeout_case {
                    0 { send "n" }
                    1 {
                        send_user "retrying...n"
                        send "n"
                    }
                    2 {
                        puts stderr "login timeout...n"
                        close
                        set done 0
                        break
                    }
                }
                incr timeout_case
            }

            "*(yes/no)?" {send "yesn"}
            "Password:" {send "$passn"}
            "password:" {send "$passn"}
            "*Permission denied*" {
                send_user "Permission deniedn"
                close
                set done 0
                break
            }
            "*Connection refused*" {
                send_user "Connection refusedn"
                close
                set done 0
                break
            }
            "*$*" {
                send -- "cd /home/top_city/dev.xxxx.com/topcity/r"
                expect "$"
                send -- "/usr/bin/svn upr"
                #sleep 70
                expect {
                    "$" {
                        set done 0
                        send_user "n"
                        send "exitn"
                    }
                }
            }
        }
    }

}

set f [open "pwd.txt" r]
while { [gets $f line] >= 0 } {
    do_ssh_login [lindex $line 0] [lindex $line 1] [lindex $line 2]
}

close $f
```