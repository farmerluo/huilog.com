---
title: Centos 5.5下搭建redmine项目管理软件
author: 阿辉
date: 2011-03-02T17:57:00+00:00
categories:
- Redmine
tags:
- Redmine
keywords:
- Redmine
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---
Redmine是一个基于web的项目管理软件，用Ruby开发。它通过“项目（Project）”的形式把成员、任务（问题）、文档、讨论以及各种形式的资源组织在一起，大家参与更新任务、文档等内容来推动项目的进度，同时系统利用时间线索（Timeline）和各种动态的报表（Report）形式来自动给成员汇报项目进度。

Redmine功能可以说是非常强大了：
<!--more-->

　　● 多项目和子项目支持

　　● 里程碑版本跟踪

　　● 可配置的用户角色控制

　　● 可配置的问题追踪系统

　　● 自动日历和甘特图绘制

　　● 支持 Blog 形式的新闻发布、Wiki 形式的文档撰写和文件管理

　　● RSS 输出和邮件通知

　　● 每个项目可以配置独立的 Wiki 和论坛模块

　　● 简单的任务时间跟踪机制

　　● 用户、项目、问题支持自定义属性

　　● 支持多种版本控制系统的绑定（SVN、CVS、Git、Mercurial 和 Darcs）

　　● 支持多 LDAP 用户认证

　　● 支持用户自注册和用户激活

　　● 多语言支持（已经内置了zh简体中文）

　　● 多数据库支持（MySQL、SQLite、PostgreSQL）

　　● 外观模版化定制（可以使用 Basecamp 、Ruby安装）

下面介绍一下在Centos 5.5下的安装，以及和nginx结合的问题。

1. 先安装一些相关库
```bash
yum groupinstall "Development Tools"
yum install zlib-devel wget openssl-devel pcre pcre-devel make gcc gcc-c++ curl-devel
```
 

2. 既然是ruby写的，ruby总是要安装的了，ruby最新的是1.9.x版了，但是最新的redmine 1.1版也只是支持ruby1.8版，所以要安装ruby 1.8.x
```bash
wget ftp://ftp.ruby-lang.org//pub/ruby/ruby-1.8.7-p334.tar.gz
tar -xvzf ruby-1.8.7-p334.tar.gz
cd ruby-1.8.7-p334

./configure
make -j3
make install
```

3. 安装rubygems，这个东东应该类似perl的module了，我是这么理解的。
```bash
wget http://production.cf.rubygems.org/rubygems/rubygems-1.5.1.tgz
tar -zxvf rubygems-1.5.1.tgz
cd rubygems-1.5.1/
ruby setup.rb
```
 
4. 用gem安装一些redmine运行所需的库
```bash
gem install rails
gem update --system
gem install rake rack
gem install i18n
gem install RedCloth
gem install fastthread --no-rdoc --no-ri
gem install mysql --no-rdoc --no-ri -- --with-mysql-dir=/usr/bin --with-mysql-lib=/usr/lib/mysql --with-mysql-include=/usr/include/mysql
```
 
5. 安装passenger，passenger是apache或nginx的一个模块，可以理解为用于apache或nginx和rails的交互的一个proxy。
```bash
gem install passenger
passenger-install-nginx-module
```

6. 下载redmine
```bash
svn co http://redmine.rubyforge.org/svn/branches/1.1-stable redmine-1.1
cp -rf redmine-1.1 /home/httpd/redmine
```

7. 建立数据库，我是用的mysql,redmine还支持其它的数据库。
```bash
mysql -h192.168.1.24 -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or g.
Your MySQL connection id is 169105
Server version: 5.0.77 Source distribution

Type 'help;' or 'h' for help. Type 'c' to clear the buffer.

mysql> create database redmine character set utf8;
Query OK, 1 row affected (0.03 sec)

mysql> quit
Bye
```

8. 配置redmine
```bash
cd /home/httpd/redmine/
cp database.yml.example database.yml
```
修改配置文件：
```bash
vi database.yml

production:
  adapter: mysql
  database: redmine
  host: 192.168.1.24
  username: root
  password: password
  encoding: utf8
```

然后跑下：
```bash
cd ..

rake generate_session_store
rake db:migrate RAILS_ENV="production"
```
如果提示哪些东西的版本不对，就安装哪些：
```bash
gem install -v=0.4.2 i18n
gem install -v=2.3.5 rails
```
然后再跑：
```bash
RAILS_ENV=production rake db:migrate
```

如果提示：
```
rake aborted!
undefined local variable or method `version_requirements' for #
```
就：
```bash
vi /home/httpd/redmine/config/environment.rb
在开头加入：
if Gem::VERSION >= "1.3.6"
    module Rails
        class GemDependency
            def requirement
                r = super
                (r == Gem::Requirement.default) ? nil : r
            end
        end
    end
end
```
启动redmine：
```bash
ruby script/server webrick -e production &
```
然后可以在浏览器内通过http://ip:3000访问redmine，redmine安装就完成了。


9. 用nginx运行redmine

虽然上面这种方式可以运行redmine,但是非常慢，经分析主要是慢在http处理这块，从上面的运行命令就可以看出来，redmine相当于是运行在一个用ruby写的web server上。能不慢嘛。。。

所以就需要用nginx运行redmine，配置也很简单：

vim nginx.conf

在http这块加入：
```
  http {
      ...
      passenger_root /usr/local/lib/ruby/gems/1.8/gems/passenger-3.0.2;
      passenger_ruby /usr/bin/ruby;
      ...
  }
```
然后加一虚拟主机：
```
    server {
      listen 80;
      server_name www.yourhost.com;
      root /home/http/redmine/public;   # 注意这边要配置redmine目录下的public目录
      passenger_enabled on;
   }
```
重启nginx，就可以通过域名访问redmine了。