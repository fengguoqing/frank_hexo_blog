---
title: Ubuntu默认启动到文本界面
categories: Linux
tags:
  - 日常记录
date: 2013-11-07 10:50:18
---

 使用虚拟机安装ubuntu之后感觉机器有点吃力，为了缓解这种压力，直接启动到文本界面，然后使用Xshell或者putty等工具登录，相当的happy。设置方法如下： 
```
    $ sudo vim /etc/default/grub
    - GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
    + GRUB_CMDLINE_LINUX_DEFAULT="text"
    $ sudo update-grub
    $ sudo reboot
```
 重启完之后就直接进入了文件界面，虚拟机的开销就会减小很多了。 