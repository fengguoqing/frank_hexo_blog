---
title: 在ubuntu上安装redmine
categories: 项目管理
date: 2014-04-20 16:33:52
tags: 
- redmine
- 教程
description:
photos:
---

最近有一个新项目需要用到项目管理工具，最后准备采用redmine，经过一系列的折腾，终于把它安装完成了，现在将安装过程分享出来，为那些遇到同样问题的同学做个参考。  

首先按照[官方网站](http://www.redmine.org/projects/redmine/wiki/RedmineInstall)的步骤来安装，但是仍旧会碰到各种各样的问题。

## 1. 下载Redmine源代码

这里利用git下载：

    git clone https://github.com/redmine/redmine

## 2. 安装配置MySQL  

已经安装过MySQL就不需要执行下面的命令：  

    sudo apt-get install mysql-server mysql-client

配置redmine数据库和用户：

    mysql -u root -p
    CREATE DATABASE redmine CHARACTER SET utf8;
    CREATE USER 'redmine'@'localhost' IDENTIFIED BY 'my_password';
    GRANT ALL PRIVILEGES ON redmine.* TO 'redmine'@'localhost';</pre>

## 3. 数据库连接配置  

首先要将redmine源码中 config/database.yml.example复制一份改名为config/database.yml。

    cp  config/database.yml.example config/database.yml

根据刚刚创建的redmine数据库修改config/database.yml

    production:
        adapter: mysql
        database: redmine
        host: localhost
        username: redmine
        password: "my_password"

## 4. 安装依赖包  

首先得安装ruby和gem,然后使用gem安装bundler，最后通过bundle根据redmine下面的Gemfile安装所有需要安装的软件包。

    sudo apt-get install ruby rubygems ruby1.8-dev ruby1.9.1-dev libmysqlclient-dev imagemagick libmagickwand-dev
    cd redmine

/* 由于有GFW的存在，需要使用国内的gem源才能下载，先删除官方源，然后添加淘宝的源 */

    gem sources -r http://rubygems.org/
    gem source -a http://ruby.taobao.org
    sudo gem install bundler -V
    bundle install --without development test

## 5. Redmine配置

    rake generate_secret_token
    RAILS_ENV=production rake db:migrate
    RAILS_ENV=production rake redmine:load_default_data

## 6. 文件系统权限设置  

在Redmine下建立文件夹并设置相应权限  

    mkdir -p tmp tmp/pdf public/plugin_assets
    sudo chmod -R 755 files log tmp public/plugin_assets

## 7. 运行测试

至此Redmine就安装完成了，现在就可以运行测试了。运行下面的命令进行测试：

    ruby script/rails server webrick -e production

运行上面的服务之后，我们就可以在浏览器中输入http://IP:3000 来测试。如果安装成功就会出现下面的网站界面：
![](http://static.oschina.net/uploads/space/2014/0420/161230_YGrG_579952.png)
初始用户名/密码：admin/admin
但是这样启动之后中断窗口是不能关闭的，如果要像服务一样启动，得添加-d参数：

    ruby script/server webrick -e production -d

如果想要关闭服务，可以通过查看该服务的PID来关闭：

    cat redmine/tmp/pids/server.pid
    kill -9 [PID]

最后在使用redmine过程中发现网络连接很慢，按理说，是局域网内的访问应该很快的，后来调查之后发现是webrick捣的鬼，改用thin就好了。

先在Gemfile文件中添加thin，然后再用bundle安装一下就可以了。

    vim Gemfile
    +#gem 'mongrel', '1.2.0.pre2'
    +gem 'thin'
    bundle install --without development test

安装之后重新启动redmine服务，访问就快很多了。  

    ruby script/rails server thin -e production -d

## 8. 邮件服务配置

邮件服务配置需要修改config/configuration.yml，我的一个可以成功发送邮件的配置是：

    # default configuration options for all environments
    default:
    # Outgoing emails configuration (see examples above)
    email_delivery:
    delivery_method: :smtp
    smtp_settings:
    address: localhost
    port: 25
    domain: example.com
    # authentication: :login
    # user_name: "redmine@example.net"
    # password: "redmine"

然后重启redmine服务，在管理>>配置>>邮件通知 中选择发送测试邮件进行测试。

## 9. 结语

在整个安装的过程中碰到了很多问题，大部分都是缺少依赖包的，在前面的安装中都已经提示出来了。
