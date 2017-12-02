---
title: 生成和打上patch的方法
categories: IT素养
tags:
  - 教程
date: 2013-12-02 13:24:16
---

在团队开发的过程中，经常需要生成patch，或者打上别人提供的patch，那么一般情况是如何操作的呢。

首先生成patch需要有两个工程，一个修改前的A工程，一个修改后的B工程。

使用linux命令diff就可以生成patch了。格式如下：
```
diff -Naur path/to/A_Project  path/to/B_Project > Project.patch
```
  **-N** 选项确保补丁文件将正确地处理已经创建或删除文件的情况。
  **-a** 将所有文件都当作文本文件处理。
  **-u** 输出每个修改前后的3行，也可以用-u5等指定输出更多上下文。
  **-r** 递归。设置后diff会将两个不同版本源代码目录中的所有对应文件全部都进行一次比较，包括子目录文件。

生成patch以后，在修改前A工程根目录下使用patch命令打上patch。
```
$ cd path/to/A_Project
$ patch -p1 < Project.patch
    -p Num 忽略几层文件夹
```
为了解释 -p 参数，需要看看如下patch文件片段：
```
--- old/modules/pcitable       Mon Sep 27 11:03:56 1999                                          +++ new/modules/pcitable       Tue Dec 19 20:05:41 2000
```
如果使用参数 -p0，那就表示从当前目录找一个叫做old的文件夹，再在它下面寻找 modules/pcitable 文件来执行patch操作。
而如果使用参数 -p1，那就表示忽略第一层目录（即不管old），从当前目录寻找 modules 的文件夹，再在它下面找pcitable。

如果要取消补丁做出的更改，恢复旧版本，在A工程的根目录下执行以下命令，A工程就会恢复成没有打patch的样子：
```
$ patch -RE -p0 < Project.patch
```
由Git管理的工程处理patch的方法：

【生成patch】

分支master 上是没修改前的，分支patch 上是修改以后的打patch的。

1. 先切换到 patch分支
> git checkout patch

2. 生成patch
> git format-patch -M master

会生成一个文件名如：0002-Release-version-1.4.6.patch 的patch文件

【合并patch】
拿到patch以后，执行下面的命令：
> git am 0002-Release-version-1.4.6.patch

如果不想直接commit，那么可以用下面这个命令打patch：
> patch -p1 < 0002-Release-version-1.4.6.patch

然后手动commit。



 