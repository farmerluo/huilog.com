---
title: svn+ssh方式svn服务器和客户端的配置
author: 阿辉
date: 2010-11-26T16:18:00+00:00
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
我们最近一个项目用的那几台服务器都是客户给的，但是管理非常严格，只给我们开了22及80端口，搞得我们更新程序只能用sftp方式，很不方便，让他们开svn端口也不肯，让我们用svn+ssh方式。

那就只能用svn+ssh方式了，不得不说svn+ssh很不方便，非常折腾。在这里记录下配置过程。

<!--more-->

一. 服务器安装和配置

1) 安装ssh server和subversion
```bash
yum install -y openssh-server subversion
```

2) 建立svn用户，此用户为ssh登录帐号，再建立用户主目录并设置权限
```bash
useradd svn
cd /home/svn
```

3) 建立 subversion repository
```bash
mkdir /var/svn-repos
svnadmin create /var/svn-repos/topcity
chown -R svn:svn /var/svn-repos/topcity
```

4) 为svnuser建立SSH公钥和私钥，svnuser就是以后用来操作的svn用户，注意它并不是linux系统用户
```bash
ssh-keygen -t rsa -b 1024 -f svnuser.key
```
这里可以输密码，也可以不输密码，如果是希望代码提交后，自动更新到运行环境的话，光用key方便点。否则建议根据提示输入密码，此时在当前目录下会生成二个文件，svnuser.key私钥和svnuser.key.pub公钥
```bash
mkdir /home/svn/.ssh
cat svnuser.key.pub >> /home/svn/.ssh/authorized_keys
chown -R svn:svn /home/svn/.ssh
```

5) 编辑 authorized_keys 文件，在相应公钥内容的开头处加入：
```bash
vi /home/svn/.ssh/authorized_keys
command=”/usr/bin/svnserve -t -r /var/svn-repos/ –tunnel-user=svnuser”,no-port-forwarding,no-pty,no-agent-forwarding,no-X11-forwarding
```
注意上述内容和原来公钥的内容应该在同一行中

6) 修改repository配置，并启用authz权限控制
```bash
vi /var/svn-repos/topcity/conf/svnserve.conf
# 在general小节中，加入三行内容
anon-access = none
auth-access = write

authz-db = /var/svn-repos/topcity/conf/authz

# 配置authz文件
vi /var/svn-repos/topcity/conf/authz
# 增加二行内容
[topcity:/]
svnuser = rw
```

7) 导入项目到svn:
```bash
svn import web svn+ssh://192.168.1.10/topcity -m “initial import”
```

8) 在svn服务器上配置自动更新(有需要的话)：
```bash
cd /var/svn-repos/topcity/hooks

cp post-commit.tmpl post-commit
chown svn:svn post-commit
chmod a+x post-commit

vi post-commit

# 把其它注释掉，在最后加上:
/var/svn-repos/topcity/hooks/svnsshup.exp
```

服务器配置OK了，下面看看客户端应该怎么使用。

二.windows客户端配置

在windows环境下，svn+ssh方式需要用到三个软件：puttygen.exe,putty.exe,TortoiseSVN。在哪下载我就不多说了，google就是。

1) 首先把上面生成的私key svnuser.key 复制到windows上，再用puttygen.exe转成putty用的key文件。

![/wp-content/uploads/baiduhi/38bcb938e64eb589d5622598.jpg](/wp-content/uploads/baiduhi/38bcb938e64eb589d5622598.jpg)

点Load选中svnuser.key,然后再点save private key，保存一个svnuser.ppk的文件。

2）在putty内配置：
Connection -> SSH -> Auth 选中刚刚转好的ppk文件
Connection -> SSH -> Data 的Auto-login username输入svn。

把Session内输入hostname和session name保存，我这边输的都是192.168.1.10。

然后打开这个Session,如果显示下面这样的提示，说明就成功了。
```
Authenticating with public key “imported-openssh-key”
Server refused to allocate pty
( success ( 1 2 ( ANONYMOUS EXTERNAL ) ( edit-pipeline svndiff1 absent-entries ) ) )
```
3) TortoiseSVN内配置：

TortoiseSVN -> Settings -> Network
选择TortoiseSVN安装目录下面的TortoisePlink.exe文件

4) checkout
先打开putty连上服务器
再用TortoiseSVN检出：`url:svn+ssh://mailto:svn@192.168.1.10/topcity`

注意url内的192.168.1.10并不是指ip地址，而是在putty内配置的session名。

能正常checkout出来就说明ok了。

三.linux客户端配置

1) 在用户目录生成.subversion：
```
svn co
```

2) 复制服务器端生成的私key过来到这目录
```
cd .subversion
cp ../svnuser.key .
```

3) 配置config文件
```bash
vi config
# 在[tunnels]内增加一行:
ssh = /usr/bin/ssh -l svn -i /home/top_city/.subversion/svnuser.key
```

4)检出：
```bash
svn checkout svn+ssh://192.168.1.10/topcity

# 更新命令是：
cd topcity
svn up
```