---
title: Citrix NetScaler VPX 导入SSL的方法
author: 阿辉
date: 2012-05-28T17:32:00+00:00
categories:
- NetScaler
tags:
- NetScaler
keywords:
- NetScaler
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
从SSL颁发机构申请证书后，他们一般会给我们三个证书，分别为：

公钥证书：xxx.cer

私钥证书：xxx.key

中级证书：WS_OV_bundle.cer (不同机构申请的不一样)

另外还有一个生成SSL证书的证书请求文件CSR，这个是用来拿cer文件的，拿了后就没什么用了。

ssl证书在nginx上的配置参考：

http://hi.baidu.com/farmerluo/item/1c83d30c9bea36f3a01034a5

<!--more-->
其中有一点需要注意的是，需要把中根证书放到公钥证书的后面：
```
cat WS_OV_bundle.cer >> xxx.cer
```

然后再按上面的文档配置一下nginx就可以了,可以通过下面这个地址去检测我们的配置是否OK：

https://sslchecker.wosign.com/default.asp

但是同样的方法（把中根证书放到公钥证书的后面，再导入公钥和私钥。），在Citrix NetScaler VPX上导入后发现不行，还是会提示中根证书没有安装。在有些浏览器上会提示安全警告。

经过多次尝试，发现在Citrix NetScaler VPX上，中根证书是需要单独导入的，然后再在公钥上link这个中根证书。


步骤大致如下：

1. 导入公钥和私钥：

![/wp-content/uploads/baiduhi/a5c27d1ed21b0ef4c7ce55e0ddc451da81cb3e6e.jpg](/wp-content/uploads/baiduhi/a5c27d1ed21b0ef4c7ce55e0ddc451da81cb3e6e.jpg)

2. 导入中根证书：

![/wp-content/uploads/baiduhi/d058ccbf6c81800aaac644b0b13533fa828b4769.jpg](/wp-content/uploads/baiduhi/d058ccbf6c81800aaac644b0b13533fa828b4769.jpg)

3. link中根证书

![/wp-content/uploads/baiduhi/5366d0160924ab18700b2c8935fae6cd7b890b6b.jpg](/wp-content/uploads/baiduhi/5366d0160924ab18700b2c8935fae6cd7b890b6b.jpg)

![/wp-content/uploads/baiduhi/03087bf40ad162d9b9b78dba11dfa9ec8a13cd3f.jpg](/wp-content/uploads/baiduhi/03087bf40ad162d9b9b78dba11dfa9ec8a13cd3f.jpg)

完成后，再用下面的工具验证一下中根证书有没有安装。
https://sslchecker.wosign.com/default.asp

参考：
http://www.entrust.net/knowledge-base/technote.cfm?tn=8186