---
title: 为你的Mail Server增加SPF记录
author: 阿辉
date: 2009-06-02T16:04:00+00:00
categories:
- Qmail
tags:
- Qmail
keywords:
- Qmail
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
什么是SPF
就是Sender Policy Framework。SPF可以防止别人伪造你来发邮件，是一个反伪造性邮件的解决方案。当你定义了你的domain name的SPF记录之后，接收邮件方会根据你的SPF记录来确定连接过来的IP地址是否被包含在SPF记录里面，如果在，则认为是一封正确的邮件，否则 则认为是一封伪造的邮件。关于更详细的信息请参考RFC4408（http://www.ietf.org/rfc/rfc4408.txt）

<!--more-->
如何增加SPF记录

非常简单，在DNS里面添加TXT记录即可。登陆http://www.openspf.org/wizard.html 在里面输入你的域名，点击Begin，然后会自动得到你域名的一些相关信息。

a 你域名的A记录，一般选择yes，因为他有可能发出邮件。

mx 一般也是yes，MX服务器会有退信等。

ptr 选择no，官方建议的。

a： 有没有其他的二级域名？比如：bbs.extmail.org和www不在一台server上，则填入bbs.extmail.org。否则清空。

mx: 一般不会再有其他的mx记录了。

ip4： 你还有没有其他的ip发信？可能你的smtp服务器是独立出来的，那么就填入你的IP地址或者网段。

include: 如果有可能通过一个isp来发信，这个有自己的SPF记录，则填入这个isp的域名，比如：sina.com

~all: 意思是除了上面的，其他的都不认可。当然是yes了。

好了，点击Continue…..

自动生成了一条SPF记录，比如extmail.org的是

`v=spf1 a mx ~all`

并且在下面告诉你如何在你的bind里面添加一条
`extmail.org. IN TXT “v=spf1 a mx ~all”`

加入你的bind，然后`ndc reload`即可。

检查一下：
`dig -t txt extmail.org`