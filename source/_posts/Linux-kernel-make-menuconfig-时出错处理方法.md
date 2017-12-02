---
title: Linux kernel make menuconfig 时出错处理方法
categories: Linux
tags:
  - linux移植
date: 2014-04-14 17:53:42
---

在Ubuntu 12.04的环境下编译最新的linux 3.14代码时出现下面的错误：
``` shell
$makemenuconfig
HOSTLDscripts/kconfig/mconf
scripts/kconfig/mconf.o:Infunction`show_help':
mconf.c:(.text+0x8a4):undefinedreferenceto`stdscr'
scripts/kconfig/lxdialog/checklist.o:Infunction`print_arrows':
checklist.c:(.text+0x41):undefinedreferenceto`wmove'
checklist.c:(.text+0x61):undefinedreferenceto`acs_map'
......
menubox.c:(.text+0x3a9):undefinedreferenceto`wrefresh'
scripts/kconfig/lxdialog/menubox.o:Infunction`print_buttons':
menubox.c:(.text+0x4ef):undefinedreferenceto`wrefresh'
collect2:ldreturned1exitstatus
make[1]:***[scripts/kconfig/mconf]Error1
make:***[menuconfig]Error2` 
```
由于缺少必要的package，所以出现了编译问题。
出现这个问题的处理方法：
``` shell
sudo apt-get install build-essential libncurses5 libncurses5-dev
```
 