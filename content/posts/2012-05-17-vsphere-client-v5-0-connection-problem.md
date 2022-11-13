---
title: vSphere Client v5.0 connection problem
author: 阿辉
date: 2012-05-17T11:34:00+00:00
categories:
- Vmware ESXI
tags:
- Vmware ESXI
keywords:
- Vmware ESXI
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
今天將vSphere client 從4.0 升級到v5.0後，發現無法連線到vSphere server.遇到的錯誤訊息如下

vSphere Client could not connect to "xx.xx.xx.xx".
An unknown connection error occurred. (The request failed because the client could not validate the server's SSL certificate. (The underlying connection was closed: Could not establish trust relationship for the SSL/TLS secure channel.))
看起來應該是certificate的問題，我也有用瀏覽器直接連線vSphere server，有看到certificate error的錯誤，也直接選擇安裝，但結果一樣，後來稍微搜尋一下，原來是憑證需要安裝到  root Trusted certificates 路徑下，所以我就在重新安裝一次，這次就自己選擇import的路進，把certifcate安裝到 root trusted certifcates下，問題即可解決，詳細的說明，可以參考下面的連結

<!--more-->
http://blogs.msdn.com/b/jpsanders/archive/2009/09/16/troubleshooting-asp-net-the-remote-certificate-is-invalid-according-to-the-validation-procedure.aspx