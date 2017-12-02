---
title: 如何架设Git服务器
categories: 项目管理
tags:
  - 教程
date: 2013-08-29 10:49:10
---

在开发的过程中往往需要一个git服务器来管理和保存代码，如何自己架设一个git服务器呢，方法很简单，这里介绍一下如何架设git服务器，搭建gitweb和push代码之后发送邮件通知组内成员。

# 1. 架设Git服务器

我们以Ubuntu为例。首先，在git服务器上创建一个名为 'git' 的用户，并为其创建一个.ssh目录。并将其权限设置为仅git用户有读写权限 
```
$ sudo adduser git
$ su git
$ cd
$ mkdir .ssh
$ chmod 700 .ssh
```
接下来，把开发者的 SSH 公钥添加到这个用户的authorized_keys文件中。假设你通过电邮收到了几个公钥并存到了临时文件里。重复一下，公钥大致看起来是这个样子：
```
$ cat /tmp/id_rsa.john.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCB007n/ww+ouN4gSLKssMxXnBOvf9LGt4L
ojG6rs6hPB09j9R/T17/x4lhJA0F3FR1rP6kYBRsWj2aThGw6HXLm9/5zytK6Ztg3RPKK+4k
Yjh6541NYsnEAZuXz0jTTyAUfrtU3Z5E003C4oxOj6H0rfIF1kKI9MAQLMdpGW1GYEIgS9Ez
Sdfd8AcCIicTDWbqLAcU4UpkaX8KyGlLwsNuuGztobF8m72ALC/nLF6JLtPofwFBlgc+myiv
O7TCUSBdLQlgMVOFq1I2uPWQOkOWQAHukEOmfjy2jctxSDBQ220ymjaNsHT4kgtZg2AYYgPq
dAv8JggJICUvax2T9va5 gsg-keypair
```
只要把它们逐个追加到authorized_keys文件尾部即可，同时将authorized_keys设置为仅git用户有读写权限。
```
$ cat /tmp/id_rsa.john.pub >> ~/.ssh/authorized_keys
$ cat /tmp/id_rsa.josie.pub >> ~/.ssh/authorized_keys
$ cat /tmp/id_rsa.jessica.pub >> ~/.ssh/authorized_keys
$ chmod 600 ~/.ssh/authorized_keys
```
现在可以用--bare选项运行git init来建立一个裸仓库，这会初始化一个不包含工作目录的仓库。
```
$ cd /opt/git
$ mkdir project.git
$ cd project.git
$ git --bare init
```
这时，Join，Josie 或者 Jessica 就可以把它加为远程仓库，推送一个分支，从而把第一个版本的项目文件上传到仓库里了。值得注意的是，每次添加一个新项目都需要通过 shell 登入主机并创建一个裸仓库目录。我们不妨以gitserver作为git用户及项目仓库所在的主机名。如果在网络内部运行该主机，并在 DNS 中设定gitserver指向该主机，那么以下这些命令都是可用的：
```
# 在 John 的电脑上
$ cd myproject
$ git init
$ git add .
$ git commit -m 'initial commit'
$ git remote add origin git@gitserver:/opt/git/project.git
$ git push origin master
```
这样，其他人的克隆和推送也一样变得很简单：
```
$ git clone git@gitserver:/opt/git/project.git
$ vim README
$ git commit -am 'fix for the README file'
$ git push origin master
```
用这个方法可以很快捷地为少数几个开发者架设一个可读写的 Git 服务。

作为一个额外的防范措施，你可以用 Git 自带的git-shell工具限制git用户的活动范围。只要把它设为git用户登入的 shell，那么该用户就无法使用普通的 bash 或者 csh 什么的 shell 程序。编辑/etc/passwd文件：
```
$ sudo vim /etc/passwd
```
在文件末尾，你应该能找到类似这样的行：
```
git:x:1000:1000::/home/git:/bin/sh
```
把bin/sh改为/usr/bin/git-shell（或者用which git-shell查看它的实际安装路径）。该行修改后的样子如下：
```
git:x:1000:1000::/home/git:/usr/bin/git-shell
```
现在git用户只能用 SSH 连接来推送和获取 Git 仓库，而不能直接使用主机 shell。尝试普通 SSH 登录的话，会看到下面这样的拒绝信息：
```
$ ssh git@gitserver
fatal: What do you think I am? A shell?
Connection to gitserver closed.
```
这里提供的方法，组内所有成员对project都有读写权限，也就是说每个分支都可以push代码，如果需要更加细致的权限控制，请使用[Gitosis](http://git-scm.com/book/zh/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-Gitosis)或者[Gitolite](http://git-scm.com/book/zh/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-Gitolite)。

# 2. 搭建Gitweb
安装gitweb之后就可以通过网站访问我们的项目了。就像[http://git.kernel.org](http://git.kernel.org)一样显示了
首先需要安装Gitweb，如果没有安装apache，那么直接安装Gitweb，也会将apache2安装的。
```
$ sudo apt-get install gitweb apache2
```
安装完成之后，我们只需要修改一下配置文件，将/etc/gitweb.conf文件中的$projectroot修改为放工程文件的目录。
```
$ vim /etc/gitweb.conf
# path to git projects (<project>.git)
$projectroot = "/opt/git";
```
至此gitweb就可以使用了，现在可以通过http://[git_server_IP]/gitweb访问了。
# 3. Push之后发送邮件通知
当组内成员push代码到服务器上之后，会自动发送邮件通知组内所有人员，该次push的具体内容是什么。具体配置方法：
一般在安装Git的时候发送邮件的脚本/usr/share/git-core/contrib/hooks/post-receive-email已经存在了，首先要修改所有者和执行权限，并且安装sendmail。
```
$ sudo chown git:git post-receive-email
$ sudo chmod 755 post-receive-email
$ sudo apt-get install sendmail
```
然后到切换到工程目录下的hooks中，添加 post-receive软链接指向 /usr/share/git-core/contrib/hooks/ post-receive-email。
```
$ cd /opt/git/project.git/hooks
$ ln -s /usr/share/git-core/contrib/hooks/post-receive-email post-receive
```
最后修改工程目录中的config文件即可。mailinglist是邮件列表，envelopesender是发件人的邮箱
```
$ vim /opt/git/project.git/config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = true

[hooks]
        mailinglist = "example@gmail.com, example2@gmail.com"     # 收件人列表
        envelopesender = project.git@example.com  # 送件人地址
        emailprefix = "[Project commit] "   # 邮件标题前缀
        showrev = "git show -C %s; echo"    # 不只显示有变化的文件，同时也显示改变的内容
```
为了使邮件显示的更清楚，还要修改一下工程目录当中的description文件，在description文件中，默认第一行是项目名称，所以要在第一行填入该项目的名称，这个在邮件中会有显示。
```
$ vim /opt/git/project.git/description
Project_A
```
我们收到邮件的标题是这样的：
```
[Project commit] Project_A branch master updated. 8811b83e1afb373cbe30d5bc25683d74ace2917c
```
有时候我们会发现启动和发送sendmail都相当慢，甚至要等两三分钟，完全不能忍，最后通过不断的测试，解决了这个问题，主要是要修改/etc/hosts。例如我的发件人的域是example.com，hostname是desktop，那么就应该这样修改：
```
127.0.0.1  example.com localhost desktop
```
改完之后重启sendmail服务:
```
service sendmail restart
```
然后你就会发现邮件发的那是刷刷滴！
至此，我们的Git服务器就已经搭建完成了，和你的小伙伴一起快乐的coding吧！
# 4. 参考链接[http://git-scm.com/book/zh/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E6%9E%B6%E8%AE%BE%E6%9C%8D%E5%8A%A1%E5%99%A8](http://git-scm.com/book/zh/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E6%9E%B6%E8%AE%BE%E6%9C%8D%E5%8A%A1%E5%99%A8) 
[http://josephj.com/entry.php?id=346](http://josephj.com/entry.php?id=346) 