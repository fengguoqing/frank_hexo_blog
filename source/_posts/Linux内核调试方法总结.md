---
title: Linux内核调试方法总结
categories: Linux
tags:
  - Linux内核调试
date: 2013-03-12 16:25:59
---

内核开发比用户空间开发更难的一个因素就是内核调试艰难。内核错误往往会导致系统宕机，很难保留出错时的现场。调试内核的关键在于你的对内核的深刻理解。

# 一 调试前的准备

在调试一个bug之前，我们所要做的准备工作有：
- 有一个被确认的bug。
- 包含这个bug的内核版本号，需要分析出这个bug在哪一个版本被引入，这个对于解决问题有极大的帮助。可以采用二分查找法来逐步锁定bug引入版本号。
- 对内核代码理解越深刻越好，同时还需要一点点运气。
- 该bug可以复现。如果能够找到复现规律，那么离找到问题的原因就不远了。
- 最小化系统。把可能产生bug的因素逐一排除掉。 

# 二 内核中的bug
内核中的bug也是多种多样的。它们的产生有无数的原因，同时表象也变化多端。从隐藏在源代码中的错误到展现在目击者面前的bug，其发作往往是一系列连锁反应的事件才可能出发的。虽然内核调试有一定的困难，但是通过你的努力和理解，说不定你会喜欢上这样的挑战。

# 三 内核调试配置选项

学习编写驱动程序要构建安装自己的内核（标准主线内核）。最重要的原因之一是：内核开发者已经建立了多项用于调试的功能。但是由于这些功能会造成额外的输出，并导致能下降，因此发行版厂商通常会禁止发行版内核中的调试功能。
## 1 内核配置</span> 

为了实现内核调试，在内核配置上增加了几项：
```
Kernel hacking  --->      
[*]   Magic SysRq key 
[*]   Kernel debugging 
[*]   Debug slab memory allocations   
[*]   Spinlock and rw-lock debugging: basic checks 
[*]   Spinlock debugging: sleep-inside-spinlock checking 
           [*]   Compile the kernel with debug info   
Device Drivers  --->   
           Generic Driver Options  ---> 
           [*]   Driver Core verbose debug messages 
General setup  ---> 
           [*]   Configure standard kernel features (for small systems)  ---> 
           [*]   Load all symbols for debugging/ksymoops
```
启用选项例如：

    slablayerdebugging（slab层调试选项）
    high-memorydebugging（高端内存调试选项）
    I/Omappingdebugging（I/O映射调试选项）
    spin-lockdebugging（自旋锁调试选项）
    stack-overflowchecking（栈溢出检查选项）
    sleep-inside-spinlockchecking（自旋锁内睡眠选项）

## 2 调试原子操作

从内核2.5开发，为了检查各类由原子操作引发的问题，内核提供了极佳的工具。内核提供了一个原子操作计数器，它可以配置成，一旦在原子操作过程中，进城进入睡眠或者做了一些可能引起睡眠的操作，就打印警告信息并提供追踪线索。所以，包括在使用锁的时候调用**schedule()**，正使用锁的时候以阻塞方式请求分配内存等，各种潜在的bug都能够被探测到。下面这些选项可以最大限度地利用该特性： 
```
CONFIG_PREEMPT = y 
CONFIG_DEBUG_KERNEL = y 
CONFIG_KLLSYMS = y 
CONFIG_SPINLOCK_SLEEP = y
```
# 四 引发bug并打印信息
## 1 BUG()和BUG_ON()
一些内核调用可以用来方便标记bug，提供断言并输出信息。最常用的两个是**BUG()**和**BUG_ON()**。
定义在<include/asm-generic>中：
``` C
#ifndef HAVE_ARCH_BUG 
#define BUG() do { 
   printk("BUG: failure at %s:%d/%s()! ", __FILE__, __LINE__, __FUNCTION__); 
   panic("BUG!");   /* 引发更严重的错误，不但打印错误消息，而且整个系统业会挂起 */ 
} while (0) 
#endif 

#ifndef HAVE_ARCH_BUG_ON 
   #define BUG_ON(condition) do { if (unlikely(condition)) BUG(); } while(0) 
#endif
```
当调用这两个宏的时候，它们会引发OOPS，导致栈的回溯和错误消息的打印。 
※ 可以把这两个调用当作断言使用，如：BUG_ON(bad_thing);
## 2 dump_stack()
有些时候，只需要在终端上打印一下栈的回溯信息来帮助你调试。这时可以使用dump_stack()。这个函数只在终端上打印寄存器上下文和函数的跟踪线索。
``` C
if (!debug_check) { 
    printk(KERN_DEBUG “provide some information…/n”); 
    dump_stack(); 
}
```
# 五 printk()
内核提供的格式化打印函数。
## 1 printk函数的健壮性
健壮性是printk最容易被接受的一个特质，几乎在任何地方，任何时候内核都可以调用它（中断上下文、进程上下文、持有锁时、多处理器处理时等）。
## 2 printk函数脆弱之处
在系统启动过程中，终端初始化之前，在某些地方是不能调用的。如果真的需要调试系统启动过程最开始的地方，有以下方法可以使用：
使用串口调试，将调试信息输出到其他终端设备。
使用early_printk()，该函数在系统启动初期就有打印能力。但它只支持部分硬件体系。
## 3 LOG等级 
printk和printf一个主要的区别就是前者可以指定一个LOG等级。内核根据这个等级来判断是否在终端上打印消息。内核把比指定等级高的所有消息显示在终端。
可以使用下面的方式指定一个LOG级别：
``` C
    printk(KERN_CRIT “Hello, world!\n”);
```
注意，第一个参数并不一个真正的参数，因为其中没有用于分隔级别（KERN_CRIT）和格式字符的逗号（,）。KERN_CRIT本身只是一个普通的字符串（事实上，它表示的是字符串 "<2>"；表 1 列出了完整的日志级别清单）。作为预处理程序的一部分，C 会自动地使用一个名为 字符串串联 的功能将这两个字符串组合在一起。组合的结果是将日志级别和用户指定的格式字符串包含在一个字符串中。
内核使用这个指定LOG级别与当前终端LOG等级console_loglevel来决定是不是向终端打印。
下面是可使用的LOG等级：
``` C
#define KERN_EMERG      "<0>"   /* system is unusable */
#define KERN_ALERT        "<1>"   /* action must be taken immediately     */ 
#define KERN_CRIT           "<2>"   /* critical conditions                                */
#define KERN_ERR            "<3>"   /* error conditions                                   */
#define KERN_WARNING  "<4>"   /* warning conditions                              */
#define KERN_NOTICE       "<5>"   /* normal but significant condition         */
#define KERN_INFO            "<6>"   /* informational                                       */
#define KERN_DEBUG        "<7>"   /* debug-level messages                       */
#define KERN_DEFAULT     "<d>"   /* Use the default kernel loglevel           */
```
注意，如果调用者未将日志级别提供给 printk，那么系统就会使用默认值 KERN_WARNING "<4>"（表示只有KERN_WARNING 级别以上的日志消息会被记录）。由于默认值存在变化，所以在使用时最好指定LOG级别。有LOG级别的一个好处就是我们可以选择性的输出LOG。比如平时我们只需要打印KERN_WARNING级别以上的关键性LOG，但是调试的时候，我们可以选择打印KERN_DEBUG等以上的详细LOG。而这些都不需要我们修改代码，只需要通过命令修改默认日志输出级别： 
``` shell
mtj@ubuntu :~$ cat /proc/sys/kernel/printk
4 4 1 7
mtj@ubuntu :~$ cat /proc/sys/kernel/printk_delay
0
mtj@ubuntu :~$ cat /proc/sys/kernel/printk_ratelimit
5
mtj@ubuntu :~$ cat /proc/sys/kernel/printk_ratelimit_burst
10
```
第一项定义了 printk API 当前使用的日志级别。这些日志级别表示了控制台的日志级别、默认消息日志级别、最小控制台日志级别和默认控制台日志级别。printk_delay 值表示的是 printk 消息之间的延迟毫秒数（用于提高某些场景的可读性）。注意，这里它的值为 0，而它是不可以通过 /proc 设置的。printk_ratelimit 定义了消息之间允许的最小时间间隔（当前定义为每 5 秒内的某个内核消息数）。消息数量是由 printk_ratelimit_burst 定义的（当前定义为 10）。如果您拥有一个非正式内核而又使用有带宽限制的控制台设备（如通过串口）， 那么这非常有用。注意，在内核中，速度限制是由调用者控制的，而不是在printk 中实现的。如果一个 printk 用户要求进行速度限制，那么该用户就需要调用printk_ratelimit 函数。
## 4 记录缓冲区
内核消息都被保存在一个LOG_BUF_LEN大小的环形队列中。 
关于LOG_BUF_LEN定义：
        ` #define __LOG_BUF_LEN (1 << CONFIG_LOG_BUF_SHIFT)` 
※ 变量CONFIG_LOG_BUF_SHIFT在内核编译时由配置文件定义，对于i386平台，其值定义如下（在linux26/arch/i386/defconfig中）：
        `CONFIG_LOG_BUF_SHIFT=18`
记录缓冲区操作：
    ① 消息被读出到用户空间时，此消息就会从环形队列中删除。
    ② 当消息缓冲区满时，如果再有printk()调用时，新消息将覆盖队列中的老消息。③ 在读写环形队列时，同步问题很容易得到解决。 
※ 这个纪录缓冲区之所以称为环形，是因为它的读写都是按照环形队列的方式进行操作的。
## 5 syslogd/klogd 
在标准的Linux系统上，用户空间的守护进程klogd从纪录缓冲区中获取内核消息，再通过syslogd守护进程把这些消息保存在系统日志文件中。klogd进程既可以从/proc/kmsg文件中，也可以通过syslog()系统调用读取这些消息。默认情况下，它选择读取/proc方式实现。klogd守护进程在消息缓冲区有新的消息之前，一直处于阻塞状态。一旦有新的内核消息，klogd被唤醒，读出内核消息并进行处理。默认情况下，处理例程就是把内核消息传给syslogd守护进程。syslogd守护进程一般把接收到的消息写入/var/log/messages文件中。不过，还是可以通过/etc/syslog.conf文件来进行配置，可以选择其他的输出文件。
![](http://static.oschina.net/uploads/img/201303/12162520_Gm0j.jpg) 

## 6 dmesg 
dmesg 命令也可用于打印和控制内核环缓冲区。这个命令使用 klogctl 系统调用来读取内核环缓冲区，并将它转发到标准输出（stdout）。这个命令也可以用来清除内核环缓冲区（使用 -c 选项），设置控制台日志级别（-n 选项），以及定义用于读取内核日志消息的缓冲区大小（-s 选项）。注意，如果没有指定缓冲区大小，那么 dmesg 会使用 klogctl 的SYSLOG_ACTION_SIZE_BUFFER 操作确定缓冲区大小。
## 7 注意
a) 虽然printk很健壮，但是看了源码你就知道，这个函数的效率很低：做字符拷贝时一次只拷贝一个字节，且去调用console输出可能还产生中断。所以如果你的驱动在功能调试完成以后做性能测试或者发布的时候千万记得尽量减少printk输出，做到仅在出错时输出少量信息。否则往console输出无用信息影响性能。
b) printk的临时缓存printk_buf只有1K，所有一次printk函数只能记录<1K的信息到log buffer，并且printk使用的“ringbuffer”.
## 8 内核printk和日志系统的总体结构
![](http://static.oschina.net/uploads/img/201303/12162606_fEh3.jpg) 
## 9 动态调试
动态调试是通过动态的开启和禁止某些内核代码来获取额外的内核信息 
首先内核选项CONFIG_DYNAMIC_DEBUG应该被设置。所有通过pr_debug()/dev_debug()打印的信息都可以动态的显示或不显示。
可以通过简单的查询语句来筛选需要显示的信息。
- 源文件名
- 函数名
- 行号（包括指定范围的行号）
- 模块名
- 格式化字符串
将要打印信息的格式写入<debugfs>/dynamic_debug/control中。
``` shell
nullarbor:~ # echo 'file svcsock.c line 1603 +p' >      <debugfs>/dynamic_debug/control 
```
参考：
1. [内核日志及printk结构浅析](http://blog.chinaunix.net/uid-26993600-id-3252420.html)
2. [Tekkaman Ninja](http://blog.chinaunix.net/uid/20543672.html)
3. [内核日志：API 及实现](http://www.ibm.com/developerworks/cn/linux/l-kernel-logging-apis/index.html)
4. [printk实现分析](http://blog.chinaunix.net/uid-22227409-id-2656917.html)
5. [dynamic-debug-howto.txt](http://www.mjmwired.net/kernel/Documentation/dynamic-debug-howto.txt)
# 六 内存调试工具
## 1 MEMWATCH
MEMWATCH 由 Johan Lindh 编写，是一个开放源代码 C 语言内存错误检测工具，您可以自己下载它。只要在代码中添加一个头文件并在 gcc 语句中定义了 MEMWATCH 之后，您就可以跟踪程序中的内存泄漏和错误了。MEMWATCH 支持ANSIC，它提供结果日志纪录，能检测双重释放（double-free）、错误释放（erroneous free）、没有释放的内存（unfreedmemory）、溢出和下溢等等。
清单 1. 内存样本（test1.c）
``` C
#include <stdlib.h>
#include <stdio.h>
#include "memwatch.h"
intmain(void)
{
    char *ptr1;
    char *ptr2;
    ptr1 = malloc(512);
    ptr2 = malloc(512);
    ptr2 = ptr1;
    free(ptr2);
    free(ptr1);
}
```
清单 1 中的代码将分配两个 512 字节的内存块，然后指向第一个内存块的指针被设定为指向第二个内存块。结果，第二个内存块的地址丢失，从而产生了内存泄漏。 
现在我们编译清单 1 的 memwatch.c。下面是一个 makefile 示例：
test1
```
gcc -DMEMWATCH -DMW_STDIO test1.c memwatch
c -o test1
```
当您运行 test1 程序后，它会生成一个关于泄漏的内存的报告。清单 2 展示了示例 memwatch.log 输出文件。
清单 2. test1 memwatch.log 文件
```
MEMWATCH 2.67 Copyright (C) 1992-1999 Johan Lindh
...
double-free: <4> test1.c(15), 0x80517b4 was freed from test1.c(14)
...
unfreed: <2> test1.c(11), 512 bytes at 0x80519e4
{FE FE FE FE FE FE FE FE FE FE FE FE ..............}
Memory usage statistics (global):
 N)umber of allocations made: 2
 L)argest memory usage : 1024
 T)otal of all alloc() calls: 1024
 U)nfreed bytes totals : 512
```
MEMWATCH 为您显示真正导致问题的行。如果您释放一个已经释放过的指针，它会告诉您。对于没有释放的内存也一样。日志结尾部分显示统计信息，包括泄漏了多少内存，使用了多少内存，以及总共分配了多少内存。
## 2 YAMD
YAMD 软件包由 Nate Eldredge 编写，可以查找 C 和 C++ 中动态的、与内存分配有关的问题。在撰写本文时，YAMD 的最新版本为 0.32。请下载 yamd-0.32.tar.gz。执行 make 命令来构建程序；然后执行 make install 命令安装程序并设置工具。
一旦您下载了 YAMD 之后，请在 test1.c 上使用它。请删除 #include memwatch.h 并对 makefile 进行如下小小的修改：
使用 YAMD 的 test1 
```
gcc -g test1.c -o test1
```
清单 3 展示了来自 test1 上的 YAMD 的输出。
清单 3. 使用 YAMD 的 test1 输出 
```
YAMD version 0.32
Executable: /usr/src/test/yamd-0.32/test1
...
INFO: Normal allocation of this block
Address 0x40025e00, size 512
...
INFO: Normal allocation of this block
Address 0x40028e00, size 512
...
INFO: Normal deallocation of this block
Address 0x40025e00, size 512
...
ERROR: Multiple freeing At
free of pointer already freed
Address 0x40025e00, size 512
...
WARNING: Memory leak
Address 0x40028e00, size 512
WARNING: Total memory leaks:
1 unfreed allocations totaling 512 bytes
*** Finished at Tue ... 10:07:15 2002
Allocated a grand total of 1024 bytes 2 allocations
Average of 512 bytes per allocation
Max bytes allocated at one time: 1024
24 K alloced internally / 12 K mapped now / 8 K max
Virtual program size is 1416 K
End.
```
YAMD 显示我们已经释放了内存，而且存在内存泄漏。让我们在清单 4 中另一个样本程序上试试 YAMD。
清单 4. 内存代码（test2.c）
``` C
#include <stdlib.h>
#include <stdio.h>
int main(void)
{
    char *ptr1;
    char *ptr2;
    char *chptr;
    int i = 1;
    ptr1 = malloc(512);
    ptr2 = malloc(512);
    chptr = (char *)malloc(512);
    for (i; i <= 512; i++) {
        chptr[i] = 'S';
    } 
    ptr2 = ptr1;
    free(ptr2);
    free(ptr1);
    free(chptr);
}
```
您可以使用下面的命令来启动 YAMD：
```
./run-yamd /usr/src/test/test2/test2
```
清单 5 显示了在样本程序 test2 上使用 YAMD 得到的输出。YAMD 告诉我们在 for 循环中有“越界（out-of-bounds）”的情况。
清单 5. 使用 YAMD 的 test2 输出 
```
Running /usr/src/test/test2/test2
Temp output to /tmp/yamd-out.1243
*********
./run-yamd: line 101: 1248 Segmentation fault (core dumped)
YAMD version 0.32
Starting run: /usr/src/test/test2/test2
Executable: /usr/src/test/test2/test2
Virtual program size is 1380 K
...
INFO: Normal allocation of this block
Address 0x40025e00, size 512
...
INFO: Normal allocation of this block
Address 0x40028e00, size 512
...
INFO: Normal allocation of this block
Address 0x4002be00, size 512
ERROR: Crash
...
Tried to write address 0x4002c000
Seems to be part of this block:
Address 0x4002be00, size 512
...
Address in question is at offset 512 (out of bounds)
Will dump core after checking heap.
Done.
```
MEMWATCH 和 YAMD 都是很有用的调试工具，它们的使用方法有所不同。对于 MEMWATCH，您需要添加包含文件memwatch.h 并打开两个编译时间标记。对于链接（link）语句，YAMD 只需要 -g 选项。
## 3 Electric Fence
多数 Linux 分发版包含一个 Electric Fence 包，不过您也可以选择下载它。Electric Fence 是一个由 Bruce Perens 编写的malloc()调试库。它就在您分配内存后分配受保护的内存。如果存在 fencepost 错误（超过数组末尾运行），程序就会产生保护错误，并立即结束。通过结合 Electric Fence 和 gdb，您可以精确地跟踪到哪一行试图访问受保护内存。ElectricFence 的另一个功能就是能够检测内存泄漏。
# 七 strace
strace 命令是一种强大的工具，它能够显示所有由用户空间程序发出的系统调用。strace 显示这些调用的参数并返回符号形式的值。strace 从内核接收信息，而且不需要以任何特殊的方式来构建内核。将跟踪信息发送到应用程序及内核开发者都很有用。在清单 6 中，分区的一种格式有错误，清单显示了 strace 的开头部分，内容是关于调出创建文件系统操作（mkfs ）的。strace 确定哪个调用导致问题出现。
清单 6. mkfs 上 strace 的开头部分
```
execve("/sbin/mkfs.jfs", ["mkfs.jfs", "-f", "/dev/test1"], &
...
open("/dev/test1", O_RDWR|O_LARGEFILE) = 4
stat64("/dev/test1", {st_mode=&, st_rdev=makedev(63, 255), ...}) = 0
ioctl(4, 0x40041271, 0xbfffe128) = -1 EINVAL (Invalid argument)
write(2, "mkfs.jfs: warning - cannot setb" ..., 98mkfs.jfs: warning -
cannot set blocksize on block device /dev/test1: Invalid argument )
 = 98
stat64("/dev/test1", {st_mode=&, st_rdev=makedev(63, 255), ...}) = 0
open("/dev/test1", O_RDONLY|O_LARGEFILE) = 5
ioctl(5, 0x80041272, 0xbfffe124) = -1 EINVAL (Invalid argument)
write(2, "mkfs.jfs: can\'t determine device"..., ..._exit(1)
 = ?
```
清单 6 显示 ioctl 调用导致用来格式化分区的 mkfs 程序失败。 ioctl BLKGETSIZE64 失败。（ BLKGET-SIZE64 在调用 ioctl的源代码中定义。) BLKGETSIZE64 ioctl 将被添加到 Linux 中所有的设备，而在这里，逻辑卷管理器还不支持它。因此，如果BLKGETSIZE64 ioctl 调用失败，mkfs 代码将改为调用较早的 ioctl 调用；这使得 mkfs 适用于逻辑卷管理器。
参考：
1. <http://www.ibm.com/developerworks/cn/linux/sdk/l-debug/index.html#resources>
# 八 OOPS 
OOPS（也称 Panic）消息包含系统错误的细节，如 CPU 寄存器的内容等。是内核告知用户有不幸发生的最常用的方式。 
内核只能发布OOPS，这个过程包括向终端上输出错误消息，输出寄存器保存的信息，并输出可供跟踪的回溯线索。通常，发送完OOPS之后，内核会处于一种不稳定的状态。
OOPS的产生有很多可能原因，其中包括内存访问越界或非法的指令等。
※ 作为内核的开发者，必定将会经常处理OOPS。
※ OOPS中包含的重要信息，对所有体系结构的机器都是完全相同的：寄存器上下文和回溯线索（回溯线索显示了导致错误发生的函数调用链）。 
## 1 ksymoops 
在 Linux 中，调试系统崩溃的传统方法是分析在发生崩溃时发送到系统控制台的 Oops 消息。一旦您掌握了细节，就可以将消息发送到 ksymoops 实用程序，它将试图将代码转换为指令并将堆栈值映射到内核符号。
※ 如：回溯线索中的地址，会通过ksymoops转化成名称可见的函数名。 
ksymoops需要几项内容：Oops 消息输出、来自正在运行的内核的 System.map 文件，还有 /proc/ksyms、vmlinux和/proc/modules。

关于如何使用 ksymoops，内核源代码 /usr/src/linux/Documentation/oops-tracing.txt 中或 ksymoops 手册页上有完整的说明可以参考。Ksymoops 反汇编代码部分，指出发生错误的指令，并显示一个跟踪部分表明代码如何被调用。

首先，将 Oops 消息保存在一个文件中以便通过 ksymoops 实用程序运行它。清单 7 显示了由安装 JFS 文件系统的 mount命令创建的 Oops 消息。 
清单 7\. ksymoops 处理后的 Oops 消息 
``` shell
ksymoops 2.4.0 on i686 2.4.17. Options used
... 15:59:37 sfb1 kernel: Unable to handle kernel NULL pointer dereference at
virtual address 0000000
... 15:59:37 sfb1 kernel: c01588fc
... 15:59:37 sfb1 kernel: *pde = 0000000
... 15:59:37 sfb1 kernel: Oops: 0000
... 15:59:37 sfb1 kernel: CPU:    0
... 15:59:37 sfb1 kernel: EIP:    0010:[jfs_mount+60/704]
... 15:59:37 sfb1 kernel: Call Trace: [jfs_read_super+287/688] 
[get_sb_bdev+563/736] [do_kern_mount+189/336] [do_add_mount+35/208]
[do_page_fault+0/1264]
... 15:59:37 sfb1 kernel: Call Trace: [<c0155d4f>]...
... 15:59:37 sfb1 kernel: [<c0106e04 ...
... 15:59:37 sfb1 kernel: Code: 8b 2d 00 00 00 00 55 ...
>>EIP; c01588fc <jfs_mount+3c/2c0> <=====
...
Trace; c0106cf3 <system_call+33/40>
Code; c01588fc <jfs_mount+3c/2c0>
00000000 <_EIP>:
Code; c01588fc <jfs_mount+3c/2c0>  <=====
  0: 8b 2d 00 00 00 00 mov 0x0,%ebp    <=====
Code; c0158902 <jfs_mount+42/2c0>
  6:  55 push %ebp
``` 
接下来，您要确定 jfs_mount 中的哪一行代码引起了这个问题。Oops 消息告诉我们问题是由位于偏移地址 3c 的指令引起的。做这件事的办法之一是对 jfs_mount.o 文件使用 objdump 实用程序，然后查看偏移地址 3c。Objdump 用来反汇编模块函数，看看您的 C 源代码会产生什么汇编指令。清单 8 显示了使用 objdump 后您将看到的内容，接着，我们查看jfs_mount 的 C 代码，可以看到空值是第 109 行引起的。偏移地址 3c 之所以很重要，是因为 Oops 消息将该处标识为引起问题的位置。
清单 8\. jfs_mount 的汇编程序清单
```
109 printk("%d\n",*ptr);
objdump jfs_mount.o
jfs_mount.o: file format elf32-i386
Disassembly of section .text:
00000000 <jfs_mount>:
  0:55 push %ebp
 ...
 2c: e8 cf 03 00 00   call   400 <chkSuper>
 31: 89 c3       mov     %eax,%ebx
 33: 58     pop     %eax
 34: 85 db       test %ebx,%ebx
 36: 0f 85 55 02 00 00 jne 291 <jfs_mount+0x291>
 3c: 8b 2d 00 00 00 00 mov 0x0,%ebp << problem line above
 42: 55 push %ebp
```
## 2 kallsyms
开发版2.5内核引入了kallsyms特性，它可以通过定义CONFIG_KALLSYMS编译选项启用。该选项可以载入内核镜像所对应的内存地址的符号名称（即函数名），所以内核可以打印解码之后的跟踪线索。相应，解码OOPS也不再需要System.map和ksymoops工具了。另外，这样做，会使内核变大些，因为地址对应符号名称必须始终驻留在内核所在内存上。
```
#cat /proc/kallsyms 
c0100240   T    _stext 
c0100240   t    run_init_process 
c0100240   T      stext 
c0100269   t    init 
    …
```
## 3 Kdump
### 3.1 Kdump 的基本概念
#### 3.1.1 什么是 kexec ？
Kexec 是实现 kdump 机制的关键，它包括 2 个组成部分：一是内核空间的系统调用 kexec_load，负责在生产内核（production kernel 或 first kernel）启动时将捕获内核（capture kernel 或 sencond kernel）加载到指定地址。二是用户空间的工具 kexec-tools，他将捕获内核的地址传递给生产内核，从而在系统崩溃的时候能够找到捕获内核的地址并运行。没有 kexec 就没有 kdump。先有 kexec 实现了在一个内核中可以启动另一个内核，才让 kdump 有了用武之地。kexec 原来的目的是为了节省 kernel 开发人员重启系统的时间，谁能想到这个“偷懒”的技术却孕育了最成功的内存转存机制呢？
#### 3.1.2 什么是 kdump ？
Kdump 的概念出现在 2005 左右，是迄今为止最可靠的内核转存机制，已经被主要的 linux™ 厂商选用。kdump是一种先进的基于 kexec 的内核崩溃转储机制。当系统崩溃时，kdump 使用 kexec 启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二内核启动用。由于 kdump 利用 kexec 启动捕获内核，绕过了 BIOS，所以第一个内核的内存得以保留。这是内核崩溃转储的本质。

kdump 需要两个不同目的的内核，生产内核和捕获内核。生产内核是捕获内核服务的对像。捕获内核会在生产内核崩溃时启动起来，与相应的 ramdisk 一起组建一个微环境，用以对生产内核下的内存进行收集和转存。
#### 3.1.3 如何使用 kdump
构建系统和 dump-capture 内核，此操作有 2 种方式可选：
1）构建一个单独的自定义转储捕获内核以捕获内核转储；
2） 或者将系统内核本身作为转储捕获内核，这就不需要构建一个单独的转储捕获内核。

方法（2）只能用于可支持可重定位内核的体系结构上；目前 i386，x86_64，ppc64 和 ia64 体系结构支持可重定位内核。构建一个可重定位内核使得不需要构建第二个内核就可以捕获转储。但是可能有时想构建一个自定义转储捕获内核以满足特定要求。
#### 3.1.4 如何访问捕获内存
在内核崩溃之前所有关于核心映像的必要信息都用 ELF 格式编码并存储在保留的内存区域中。ELF 头所在的物理地址被作为命令行参数（fcorehdr=）传递给新启动的转储内核。

在 i386 体系结构上，启动的时候需要使用物理内存开始的 640K，而不管操作系统内核转载在何处。因此，这个640K 的区域在重新启动第二个内核的时候由 kexec 备份。

在第二个内核中，“前一个系统的内存”可以通过两种方式访问：
1） 通过 /dev/oldmem 这个设备接口。
一个“捕捉”设备可以使用“raw”（裸的）方式 “读”这个设备文件并写出到文件。这是关于内存的 “裸”的数据转储，同时这些分析 / 捕捉工具应该足够“智能”从而可以知道从哪里可以得到正确的信息。ELF 文件头（通过命令行参数传递过来的 elfcorehdr）可能会有帮助。
2） 通过 /proc/vmcore。
这个方式是将转储输出为一个 ELF 格式的文件，并且可以使用一些文件拷贝命令（比如 cp，scp 等）将信息读出来。同时，gdb 可以在得到的转储文件上做一些调试（有限的）。这种方式保证了内存中的页面都以正确的途径被保存 ( 注意内存开始的 640K 被重新映射了 )。
#### 3.1.5 kdump 的优势
1）高可靠性
崩溃转储数据可从一个新启动内核的上下文中获取，而不是从已经崩溃内核的上下文。
2）多版本支持
LKCD(Linux Kernel Crash Dump)，netdump，diskdump 已被纳入 LDPs(Linux Documen-tation Project) 内核。SUSE 和 RedHat 都对 kdump 有技术支持。
### 3.2 Kdump 实现流程
![图 1. RHEL6.2 执行流程](http://static.oschina.net/uploads/img/201303/13095818_2Eau.jpg "图 1. RHEL6.2 执行流程")
![图 2\. sles11 执行流程](http://static.oschina.net/uploads/img/201303/13095819_vtjb.jpg "图 2. sles11 执行流程")
### 3.3 配置 kdump
#### 3.3.1 安装软件包和实用程序
Kdump 用到的各种工具都在 kexec-tools 中。kernel-debuginfo 则是用来分析 vmcore 文件。从 rhel5 开始，kexec-tools 已被默认安装在发行版。而 novell 也在 sles10 发行版中把 kdump 集成进来。所以如果使用的是rhel5 和 sles10 之后的发行版，那就省去了安装 kexec-tools 的步骤。而如果需要调试 kdump 生成的 vmcore文件，则需要手动安装 kernel-debuginfo 包。检查安装包操作：
#### 3.3.2参数相关设置
```
 uli13lp1:/ # rpm -qa|grep kexec 
 kexec-tools-2.0.0-53.43.10 
 uli13lp1:/ # rpm -qa 'kernel*debuginfo*'
 kernel-default-debuginfo-3.0.13-0.27.1 
 kernel-ppc64-debuginfo-3.0.13-0.27.1
```
系统内核设置选项和转储捕获内核配置选择在《使用 Crash 工具分析 Linux dump 文件》一文中已有说明，在此不再赘述。仅列出内核引导参数设置以及配置文件设置。 
1） 修改内核引导参数，为启动捕获内核预留内存
通过下面的方法来配置 kdump 使用的内存大小。添加启动参数"crashkernel=Y@X"，这里，Y 是为 kdump 捕捉内核保留的内存，X 是保留部分内存的开始位置。

* 对于 i386 和 x86_64, 编辑 /etc/grub.conf, 在内核行的最后添加"crashkernel=128M" 。
* 对于 ppc64，在 /etc/yaboot.conf 最后添加"crashkernel=128M"。 

在 ia64, 编辑 /etc/elilo.conf，添加"crashkernel=256M"到内核行。
2） kdump 配置文件
kdump 的配置文件是 /etc/kdump.conf（RHEL6.2）；/etc/sysconfig/kdump(SLES11 sp2)。每个文件头部都有选项说明，可以根据使用需求设置相应的选项。
### 3.3.3 启动 kdump 服务
在设置了预留内存后，需要重启机器，否则 kdump 是不可使用的。启动 kdump 服务：

Rhel6.2：
```
# chkconfig kdump on 
# service kdump status  
Kdump is operational  
# service kdump start
```
SLES11SP2：
```
 # chkconfig boot.kdump on 
 # service boot.kdump start
```
#### 3.3.4 测试配置是否有效
可以通过 kexec 加载内核镜像，让系统准备好去捕获一个崩溃时产生的 vmcore。可以通过 sysrq 强制系统崩溃。
`# echo c > /proc/sysrq-trigger`
这造成内核崩溃，如配置有效，系统将重启进入 kdump 内核，当系统进程进入到启动 kdump 服务的点时，vmcore 将会拷贝到你在 kdump 配置文件中设置的位置。RHEL 的缺省目录是 : /var/crash；SLES 的缺省目录是 : /var/log/dump。然后系统重启进入到正常的内核。一旦回复到正常的内核，就可以在上述的目录下发现 vmcore 文件，即内存转储文件。可以使用之前安装的 kernel-debuginfo 中的 crash 工具来进行分析（crash 的更多详细用法将在本系列后面的文章中有介绍）。
```
 # crash /usr/lib/debug/lib/modules/2.6.17-1.2621.el5/vmlinux 
 /var/crash/2006-08-23-15:34/vmcore 
 crash> bt
``` 
### 3.4 载入“转储捕获”内核
    需要引导系统内核时，可使用如下步骤和命令载入“转储捕获”内核：
```
kexec -p <dump-capture-kernel> \ 
           --initrd=<initrd-for-dump-capture-kernel> --args-linux \ 
           --append="root=<root-dev> init 1 irqpoll"
```
装载转储捕捉内核的注意事项：

*   转储捕捉内核应当是一个 vmlinux 格式的映像（即是一个未压缩的 ELF 映像文件），而不能是 bzImage 格式；
*   默认情况下，ELF 文件头采用 ELF64 格式存储以支持那些拥有超过 4GB 内存的系统。但是可以指定“--elf32-core-headers”标志以强制使用 ELF32 格式的 ELF 文件头。这个标志是有必要注意的，一个重要的原因就是：当前版本的 GDB 不能在一个 32 位系统上打开一个使用 ELF64 格式的 vmcore 文件。ELF32 格式的文件头不能使用在一个“没有物理地址扩展”（non-PAE）的系统上（即：少于 4GB 内存的系统）;
*   一个“irqpoll”的启动参数可以减低由于在“转储捕获内核”中使用了“共享中断”技术而导致出现驱动初始化失败这种情况发生的概率 ;
*   必须指定 <root-dev>，指定的格式是和要使用根设备的名字。具体可以查看 mount 命令的输出；“init 1”这个命令将启动“转储捕捉内核”到一个没有网络支持的单用户模式。如果你希望有网络支持，那么使用“init 3”。 

### 3.5 后记

Kdump 是一个强大的、灵活的内核转储机制，能够在生产内核上下文中执行捕获内核是非常有价值的。本文仅介绍在 RHEL6.2 和 SLES11 中如何配置 kdump。望抛砖引玉，对阅读本文的读者有益。
参考：

1. [kallsyms的分析](http://linux.chinaunix.net/jh/4/1013999.html) 
2.  [深入探索 Kdump](http://www.ibm.com/developerworks/cn/linux/l-cn-kdump1/index.html) 

# 九 KGDB 
kgdb提供了一种使用 gdb调试 Linux 内核的机制。使用KGDB可以象调试普通的应用程序那样，在内核中进行设置断点、检查变量值、单步跟踪程序运行等操作。使用KGDB调试时需要两台机器，一台作为开发机（Development Machine）,另一台作为目标机（Target Machine），两台机器之间通过串口或者以太网口相连。串口连接线是一根RS-232接口的电缆，在其内部两端的第2脚（TXD）与第3脚（RXD）交叉相连，第7脚（接地脚）直接相连。调试过程中，被调试的内核运行在目标机上，gdb调试器运行在开发机上。
目前，kgdb发布支持i386、x86_64、32-bit PPC、SPARC等几种体系结构的调试器。
## 1 kgdb的调试原理 
安装kgdb调试环境需要为Linux内核应用kgdb补丁，补丁实现的gdb远程调试所需要的功能包括命令处理、陷阱处理及串口通讯3个主要的部分。kgdb补丁的主要作用是在Linux内核中添加了一个调试Stub。调试Stub是Linux内核中的一小段代码，提供了运行gdb的开发机和所调试内核之间的一个媒介。gdb和调试stub之间通过gdb串行协议进行通讯。gdb串行协议是一种基于消息的ASCII码协议，包含了各种调试命令。当设置断点时，kgdb负责在设置断点的指令前增加一条trap指令，当执行到断点时控制权就转移到调试stub中去。此时，调试stub的任务就是使用远程串行通信协议将当前环境传送给gdb，然后从gdb处接受命令。gdb命令告诉stub下一步该做什么，当stub收到继续执行的命令时，将恢复程序的运行环境，把对CPU的控制权重新交还给内核
![](http://static.oschina.net/uploads/img/201303/12162607_r9Pn.jpg)
## 2 Kgdb的安装与设置
下面我们将以Linux 2.6.7内核为例详细介绍kgdb调试环境的建立过程。
### 2.1 软硬件准备
以下软硬件配置取自笔者进行试验的系统配置情况：
![](http://static.oschina.net/uploads/img/201303/12162608_geFn.jpg)
kgdb补丁的版本遵循如下命名模式：Linux-A-kgdb-B，其中A表示Linux的内核版本号，B为kgdb的版本号。以试验使用的kgdb补丁为例，linux内核的版本为linux-2.6.7，补丁版本为kgdb-2.2。
物理连接好串口线后，使用以下命令来测试两台机器之间串口连接情况，stty命令可以对串口参数进行设置：
在development机上执行：
在target机上执行：
```
stty ispeed 115200 ospeed 115200 -F /dev/ttyS0
```
在developement机上执行：

```
echo hello > /dev/ttyS0
```
在target机上执行： 
```
cat /dev/ttyS0
```
如果串口连接没问题的话在将在target机的屏幕上显示"hello"。 

### 2.2  安装与配置

下面我们需要应用kgdb补丁到Linux内核，设置内核选项并编译内核。这方面的资料相对较少，笔者这里给出详细的介绍。下面的工作在开发机（developement）上进行，以上面介绍的试验环境为例，某些具体步骤在实际的环境中可能要做适当的改动： 


#### I、内核的配置与编译

```
[root@lisl tmp]# tar -jxvf linux-2.6.7.tar.bz2
[root@lisl tmp]#tar -jxvf linux-2.6.7-kgdb-2.2.tar.tar
[root@lisl tmp]#cd inux-2.6.7
```
请参照目录补丁包中文件README给出的说明，执行对应体系结构的补丁程序。由于试验在i386体系结构上完成，所以只需要安装一下补丁：**core-lite.patch、i386-lite.patch、8250.patch、eth.patch、core.patch、i386.patch**。应用补丁文件时，请遵循kgdb软件包内series文件所指定的顺序，否则可能会带来预想不到的问题。eth.patch文件是选择以太网口作为调试的连接端口时需要运用的补丁。 
应用补丁的命令如下所示： 
```
[root@lisl tmp]#patch -p1 <../linux-2.6.7-kgdb-2.2/core-lite.patch
```
如果内核正确，那么应用补丁时应该不会出现任何问题（不会产生*.rej文件）。为Linux内核添加了补丁之后，需要进行内核的配置。内核的配置可以按照你的习惯选择配置Linux内核的任意一种方式。 
```
[root@lisl tmp]#make menuconfig
```
在内核配置菜单的Kernel hacking选项中选择kgdb调试项，例如： 
```
[*] KGDB: kernel debugging with remote gdb
      Method for KGDB communication (KGDB: On generic serial port (8250))  --->  
 [*] KGDB: Thread analysis 
 [*] KGDB: Console messages through gdb
[root@lisl tmp]#make
```
编译内核之前请注意Linux目录下Makefile中的优化选项，默认的Linux内核的编译都以-O2的优化级别进行。在这个优化级别之下，编译器要对内核中的某些代码的执行顺序进行改动，所以在调试时会出现程序运行与代码顺序不一致的情况。可以把Makefile中的-O2选项改为-O,但不可去掉-O，否则编译会出问题。为了使编译后的内核带有调试信息，注意在编译内核的时候需要加上-g选项。 
不过，当选择"Kernel debugging->Compile the kernel with debug info"选项后配置系统将自动打开调试选项。另外，选择"kernel debugging with remote gdb"后，配置系统将自动打开"Compile the kernel with debug info"选项。
内核编译完成后，使用scp命令进行将相关文件拷贝到target机上(当然也可以使用其它的网络工具，如rcp)。 
```
[root@lisl tmp]#scp arch/i386/boot/bzImage root@192.168.6.13:/boot/vmlinuz-2.6.7-kgdb
[root@lisl tmp]#scp System.map root@192.168.6.13:/boot/System.map-2.6.7-kgdb
```
如果系统启动使所需要的某些设备驱动没有编译进内核的情况下，那么还需要执行如下操作： 
```
[root@lisl tmp]#mkinitrd /boot/initrd-2.6.7-kgdb 2.6.7
[root@lisl tmp]#scp initrd-2.6.7-kgdb root@192.168.6.13:/boot/ initrd-2.6.7-kgdb
```
#### II、kgdb的启动 
在将编译出的内核拷贝的到target机器之后，需要配置系统引导程序，加入内核的启动选项。以下是kgdb内核引导参数的说明： 
![](http://static.oschina.net/uploads/img/201303/12162609_GxMC.jpg)
如表中所述，在kgdb 2.0版本之后内核的引导参数已经与以前的版本有所不同。使用grub引导程序时，直接将kgdb参数作为内核vmlinuz的引导参数。下面给出引导器的配置示例。 
```
title 2.6.7 kgdb
root (hd0,0)
kernel /boot/vmlinuz-2.6.7-kgdb ro root=/dev/hda1 kgdbwait kgdb8250=1,115200
```
在使用lilo作为引导程序时，需要把kgdb参放在由append修饰的语句中。下面给出使用lilo作为引导器时的配置示例。 
```
image=/boot/vmlinuz-2.6.7-kgdb
label=kgdb
   read-only
   root=/dev/hda3
append="gdb gdbttyS=1 gdbbaud=115200"
```
保存好以上配置后重新启动计算机，选择启动带调试信息的内核，内核将在短暂的运行后在创建init内核线程之前停下来，打印出以下信息，并等待开发机的连接。 
> Waiting for connection from remote gdb... 

在开发机上执行： 
```
gdb
file vmlinux
set remotebaud 115200
target remote /dev/ttyS0
```
其中vmlinux是指向源代码目录下编译出来的Linux内核文件的链接，它是没有经过压缩的内核文件，gdb程序从该文件中得到各种符号地址信息。 
这样，就与目标机上的kgdb调试接口建立了联系。一旦建立联接之后，对Linux内的调试工作与对普通的运用程序的调试就没有什么区别了。任何时候都可以通过键入ctrl+c打断目标机的执行，进行具体的调试工作。 
在kgdb 2.0之前的版本中，编译内核后在arch/i386/kernel目录下还会生成可执行文件gdbstart。将该文件拷贝到target机器的/boot目录下，此时无需更改内核的启动配置文件，直接使用命令： 
```
[root@lisl boot]#gdbstart -s 115200 -t /dev/ttyS0
```
可以在KGDB内核引导启动完成后建立开发机与目标机之间的调试联系。 

### 2.3  通过网络接口进行调试 
kgdb也支持使用以太网接口作为调试器的连接端口。在对Linux内核应用补丁包时，需应用eth.patch补丁文件。配置内核时在Kernel hacking中选择kgdb调试项，配置kgdb调试端口为以太网接口，例如： 
```
[*]KGDB: kernel debugging with remote gdb
Method for KGDB communication (KGDB: On ethernet)  ---> 
( ) KGDB: On generic serial port (8250)
(X) KGDB: On ethernet
```
另外使用eth0网口作为调试端口时，grub.list的配置如下： 
```
title 2.6.7 kgdb
root (hd0,0)
kernel /boot/vmlinuz-2.6.7-kgdb ro root=/dev/hda1 kgdbwait kgdboe=@192.168.5.13/,@192.168. 6.13/
```
其他的过程与使用串口作为连接端口时的设置过程相同。 
注意：尽管可以使用以太网口作为kgdb的调试端口，使用串口作为连接端口更加简单易行，kgdb项目组推荐使用串口作为调试端口。 

### 2.4  模块的调试方法 
内核可加载模块的调试具有其特殊性。由于内核模块中各段的地址是在模块加载进内核的时候才最终确定的，所以develop机的gdb无法得到各种符号地址信息。所以，使用kgdb调试模块所需要解决的一个问题是，需要通过某种方法获得可加载模块的最终加载地址信息，并把这些信息加入到gdb环境中。 
#### I、在Linux 2.4内核中的内核模块调试方法 
在Linux2.4.x内核中，可以使用insmod -m命令输出模块的加载信息，例如： 
```
[root@lisl tmp]# insmod -m hello.ko >modaddr
```
查看模块加载信息文件modaddr如下： 
```
.this           00000060  c88d8000  2**2
.text           00000035  c88d8060  2**2
.rodata         00000069  c88d80a0  2**5
……
.data           00000000  c88d833c  2**2
.bss            00000000  c88d833c  2**2
……
```
在这些信息中，我们关心的只有4个段的地址:.text、.rodata、.data、.bss。在development机上将以上地址信息加入到gdb中,这样就可以进行模块功能的测试了。 
```
(gdb) Add-symbol-file hello.o 0xc88d8060 -s .data 0xc88d80a0 -s .rodata 0xc88d80a0 -s .bss 0x c88d833c
```
这种方法也存在一定的不足，它不能调试模块初始化的代码，因为此时模块初始化代码已经执行过了。而如果不执行模块的加载又无法获得模块插入地址，更不可能在模块初始化之前设置断点了。对于这种调试要求可以采用以下替代方法。 
在target机上用上述方法得到模块加载的地址信息，然后再用rmmod卸载模块。在development机上将得到的模块地址信息导入到gdb环境中，在内核代码的调用初始化代码之前设置断点。这样，在target机上再次插入模块时，代码将在执行模块初始化之前停下来，这样就可以使用gdb命令调试模块初始化代码了。 
另外一种调试模块初始化函数的方法是：当插入内核模块时，内核模块机制将调用函数sys_init_module(kernel/modle.c)执行对内核模块的初始化，该函数将调用所插入模块的初始化函数。程序代码片断如下： 
```
…… ……
if (mod->init != NULL)
ret = mod->init();
…… ……
```
在该语句上设置断点，也能在执行模块初始化之前停下来。 

#### II、在Linux 2.6.x内核中的内核模块调试方法

Linux 2.6之后的内核中，由于module-init-tools工具的更改，insmod命令不再支持-m参数，只有采取其他的方法来获取模块加载到内核的地址。通过分析ELF文件格式，我们知道程序中各段的意义如下： 
**.text（代码段）**：用来存放可执行文件的操作指令，也就是说是它是可执行程序在内存种的镜像。 
**.data（数据段）**：数据段用来存放可执行文件中已初始化全局变量，也就是存放程序静态分配的变量和全局变量。 
**.bss（BSS段）**：BSS段包含了程序中未初始化全局变量，在内存中 bss段全部置零。 
**.rodata（只读段）**：该段保存着只读数据，在进程映象中构造不可写的段。 
通过在模块初始化函数中放置一下代码，我们可以很容易地获得模块加载到内存中的地址。 
``` C
……
int bss_var;
static int hello_init(void)
{
printk(KERN_ALERT "Text location .text(Code Segment):%p\n",hello_init);
static int data_var=0;
printk(KERN_ALERT "Data Location .data(Data Segment):%p\n",&data_var);
printk(KERN_ALERT "BSS Location: .bss(BSS Segment):%p\n",&bss_var);
……
}
Module_init(hello_init);
```
这里，通过在模块的初始化函数中添加一段简单的程序，使模块在加载时打印出在内核中的加载地址。.rodata段的地址可以通过执行命令readelf -e hello.ko，取得.rodata在文件中的偏移量并加上段的align值得出。 
为了使读者能够更好地进行模块的调试，kgdb项目还发布了一些脚本程序能够自动探测模块的插入并自动更新gdb中模块的符号信息。这些脚本程序的工作原理与前面解释的工作过程相似，更多的信息请阅读参考资料[4]。 

### 2.5  硬件断点 
kgdb提供对硬件调试寄存器的支持。在kgdb中可以设置三种硬件断点：执行断点（Execution Breakpoint）、写断点（Write Breakpoint）、访问断点（Access Breakpoint）但不支持I/O访问的断点。 目前，kgdb对硬件断点的支持是通过宏来实现的，最多可以设置4个硬件断点，这些宏的用法如下： 
![](http://static.oschina.net/uploads/img/201303/12162610_ODYA.jpg)
在有些情况下，硬件断点的使用对于内核的调试是非常方便的。 

## 3  在VMware中搭建调试环境

kgdb调试环境需要使用两台微机分别充当development机和target机，使用VMware后我们只使用一台计算机就可以顺利完成kgdb调试环境的搭建。以windows下的环境为例，创建两台虚拟机，一台作为开发机，一台作为目标机。 

### 3.1  虚拟机之间的串口连接 
虚拟机中的串口连接可以采用两种方法。一种是指定虚拟机的串口连接到实际的COM上，例如开发机连接到COM1，目标机连接到COM2，然后把两个串口通过串口线相连接。另一种更为简便的方法是：在较高一些版本的VMware中都支持把串口映射到命名管道，把两个虚拟机的串口映射到同一个命名管道。例如，在两个虚拟机中都选定同一个命名管道 \\.\pipe\com_1,指定target机的COM口为server端，并选择"The other end is a virtual machine"属性；指定development机的COM口端为client端，同样指定COM口的"The other end is a virtual machine"属性。对于IO mode属性，在target上选中"Yield CPU on poll"复选择框，development机不选。这样，可以无需附加任何硬件，利用虚拟机就可以搭建kgdb调试环境。 即降低了使用kgdb进行调试的硬件要求，也简化了建立调试环境的过程。 
![](http://static.oschina.net/uploads/img/201303/12162610_WfIP.jpg)
### 3.2  VMware的使用技巧

VMware虚拟机是比较占用资源的，尤其是象上面那样在Windows中使用两台虚拟机。因此，最好为系统配备512M以上的内存，每台虚拟机至少分配128M的内存。这样的硬件要求，对目前主流配置的PC而言并不是过高的要求。出于系统性能的考虑，在VMware中尽量使用字符界面进行调试工作。同时，Linux系统默认情况下开启了sshd服务，建议使用SecureCRT登陆到Linux进行操作，这样可以有较好的用户使用界面。 

### 3.3  在Linux下的虚拟机中使用kgdb 
对于在Linux下面使用VMware虚拟机的情况，笔者没有做过实际的探索。从原理上而言，只需要在Linux下只要创建一台虚拟机作为target机，开发机的工作可以在实际的Linux环境中进行，搭建调试环境的过程与上面所述的过程类似。由于只需要创建一台虚拟机，所以使用Linux下的虚拟机搭建kgdb调试环境对系统性能的要求较低。（vmware已经推出了Linux下的版本）还可以在development机上配合使用一些其他的调试工具，例如功能更强大的cgdb、图形界面的DDD调试器等，以方便内核的调试工作。 
![](http://static.oschina.net/uploads/img/201303/12162611_uyPK.jpg)

## 4  kgdb的一些特点和不足

使用kgdb作为内核调试环境最大的不足在于对kgdb硬件环境的要求较高，必须使用两台计算机分别作为target和development机。尽管使用虚拟机的方法可以只用一台PC即能搭建调试环境，但是对系统其他方面的性能也提出了一定的要求，同时也增加了搭建调试环境时复杂程度。另外，kgdb内核的编译、配置也比较复杂，需要一定的技巧，笔者当时做的时候也是费了很多周折。当调试过程结束后时，还需要重新制作所要发布的内核。使用kgdb并不能进行全程调试，也就是说kgdb并不能用于调试系统一开始的初始化引导过程。 
不过，kgdb是一个不错的内核调试工具，使用它可以进行对内核的全面调试，甚至可以调试内核的中断处理程序。如果在一些图形化的开发工具的帮助下，对内核的调试将更方便。 

参考： 
[透过虚拟化技术体验kgdb](http://blog.linux.org.tw/~jserv/archives/002045.html)
[Linux 系统内核的调试](http://www.ibm.com/developerworks/cn/linux/l-kdb/index.html)
[Debugging The Linux Kernel Using Gdb](http://elinux.org/Debugging_The_Linux_Kernel_Using_Gdb)

# 十  使用SkyEye构建Linux内核调试环境

SkyEye是一个开源软件项目（OPenSource Software）,SkyEye项目的目标是在通用的Linux和Windows平台上模拟常见的嵌入式计算机系统。SkyEye实现了一个指令级的硬件模拟平台，可以模拟多种嵌入式开发板，支持多种CPU指令集。SkyEye 的核心是 GNU 的 gdb 项目，它把gdb和 ARM Simulator很好地结合在了一起。加入ARMulator 的功能之后，它就可以来仿真嵌入式开发板，在它上面不仅可以调试硬件驱动，还可以调试操作系统。Skyeye项目目前已经在嵌入式系统开发领域得到了很大的推广。 

## 1  SkyEye的安装和μcLinux内核编译

### 1.1  SkyEye的安装 
SkyEye的安装不是本文要介绍的重点，目前已经有大量的资料对此进行了介绍。有关SkyEye的安装与使用的内容请查阅参考资料[11]。由于skyeye面目主要用于嵌入式系统领域，所以在skyeye上经常使用的是μcLinux系统，当然使用Linux作为skyeye上运行的系统也是可以的。由于介绍μcLinux 2.6在skyeye上编译的相关资料并不多，所以下面进行详细介绍。 

### 1.2  μcLinux 2.6.x的编译 
要在SkyEye中调试操作系统内核，首先必须使被调试内核能在SkyEye所模拟的开发板上正确运行。因此，正确编译待调试操作系统内核并配置SkyEye是进行内核调试的第一步。下面我们以SkyEye模拟基于Atmel AT91X40的开发板，并运行μcLinux 2.6为例介绍SkyEye的具体调试方法。 

#### I、安装交叉编译环境 
先安装交叉编译器。尽管在一些资料中说明使用工具链arm-elf-tools-20040427.sh ,但是由于arm-elf-xxx与arm-linux-xxx对宏及链接处理的不同，经验证明使用arm-elf-xxx工具链在链接vmlinux的最后阶段将会出错。所以这里我们使用的交叉编译工具链是：arm-uclinux-tools-base-gcc3.4.0-20040713.sh，关于该交叉编译工具链的下载地址请参见[6]。注意以下步骤最好用root用户来执行。 
```
[root@lisl tmp]#chmod +x  arm-uclinux-tools-base-gcc3.4.0-20040713.sh
[root@lisl tmp]#./arm-uclinux-tools-base-gcc3.4.0-20040713.sh
```
安装交叉编译工具链之后，请确保工具链安装路径存在于系统PATH变量中。 

#### II、制作μcLinux内核 
得到μcLinux发布包的一个最容易的方法是直接访问uClinux.org站点[7]。该站点发布的内核版本可能不是最新的，但你能找到一个最新的μcLinux补丁以及找一个对应的Linux内核版本来制作一个最新的μcLinux内核。这里，将使用这种方法来制作最新的μcLinux内核。目前（笔者记录编写此文章时），所能得到的发布包的最新版本是uClinux-dist.20041215.tar.gz。 
下载uClinux-dist.20041215.tar.gz，文件的下载地址请参见[7]。 
下载linux-2.6.9-hsc0.patch.gz，文件的下载地址请参见[8]。 
下载linux-2.6.9.tar.bz2，文件的下载地址请参见[9]。 
现在我们得到了整个的linux-2.6.9源代码，以及所需的内核补丁。请准备一个有2GB空间的目录里来完成以下制作μcLinux内核的过程。 
```
[root@lisl tmp]# tar -jxvf uClinux-dist-20041215.tar.bz2
[root@lisl uClinux-dist]# tar -jxvf  linux-2.6.9.tar.bz2
[root@lisl uClinux-dist]# gzip -dc linux-2.6.9-hsc0.patch.gz | patch -p0
```
或者使用：
```
[root@lisl uClinux-dist]# gunzip linux-2.6.9-hsc0.patch.gz 
[root@lisl uClinux-dist]patch -p0 < linux-2.6.9-hsc0.patch
```
执行以上过程后，将在linux-2.6.9/arch目录下生成一个补丁目录－armnommu。删除原来μcLinux目录里的linux-2.6.x(即那个linux-2.6.9-uc0)，并将我们打好补丁的Linux内核目录更名为linux-2.6.x。 
```
[root@lisl uClinux-dist]# rm -rf linux-2.6.x/
[root@lisl uClinux-dist]# mv linux-2.6.9 linux-2.6.x
```
III、配置和编译μcLinux内核 
因为只是出于调试μcLinux内核的目的，这里没有生成uClibc库文件及romfs.img文件。在发布μcLinux时，已经预置了某些常用嵌入式开发板的配置文件，因此这里直接使用这些配置文件，过程如下： 
```
[root@lisl uClinux-dist]# cd linux-2.6.x
[root@lisl linux-2.6.x]#make ARCH=armnommu CROSS_COMPILE=arm-uclinux- atmel_deconfig
```
atmel_deconfig文件是μcLinux发布时提供的一个配置文件，存放于目录linux-2.6.x /arch/armnommu/configs/中。 
```
[root@lisl linux-2.6.x]#make ARCH=armnommu CROSS_COMPILE=arm-uclinux- oldconfig
```
下面编译配置好的内核： 
```
[root@lisl linux-2.6.x]# make ARCH=armnommu CROSS_COMPILE=arm-uclinux- v=1
```

一般情况下，编译将顺利结束并在Linux-2.6.x/目录下生成未经压缩的μcLinux内核文件vmlinux。需要注意的是为了调试μcLinux内核，需要打开内核编译的调试选项-g，使编译后的内核带有调试信息。打开编译选项的方法可以选择： 
"Kernel debugging->Compile the kernel with debug info"后将自动打开调试选项。也可以直接修改linux-2.6.x目录下的Makefile文件，为其打开调试开关。方法如下：。 
> CFLAGS  += -g

最容易出现的问题是找不到arm-uclinux-gcc命令的错误，主要原因是PATH变量中没有 包含arm-uclinux-gcc命令所在目录。在arm-linux-gcc的缺省安装情况下，它的安装目录是/root/bin/arm-linux-tool/，使用以下命令将路径加到PATH环境变量中。 
```
Export PATH＝$PATH:/root/bin/arm-linux-tool/bin
```
#### IV、根文件系统的制作 
Linux内核在启动的时的最后操作之一是加载根文件系统。根文件系统中存放了嵌入式 系统使用的所有应用程序、库文件及其他一些需要用到的服务。出于文章篇幅的考虑，这里不打算介绍根文件系统的制作方法，读者可以查阅一些其他的相关资料。值得注意的是，由配置文件skyeye.conf指定了装载到内核中的根文件系统。 

## 2  使用SkyEye调试

编译完μcLinux内核后，就可以在SkyEye中调试该ELF执行文件格式的内核了。前面已经说过利用SkyEye调试内核与使用gdb调试运用程序的方法相同。 
需要提醒读者的是，SkyEye的配置文件－skyeye.conf记录了模拟的硬件配置和模拟执行行为。该配置文件是SkyEye系统中一个及其重要的文件，很多错误和异常情况的发生都和该文件有关。在安装配置SkyEye出错时，请首先检查该配置文件然后再进行其他的工作。此时，所有的准备工作已经完成，就可以进行内核的调试工作了。 

## 3  使用SkyEye调试内核的特点和不足

在SkyEye中可以进行对Linux系统内核的全程调试。由于SkyEye目前主要支持基于ARM内核的CPU，因此一般而言需要使用交叉编译工具编译待调试的Linux系统内核。另外，制作SkyEye中使用的内核编译、配置过程比较复杂、繁琐。不过，当调试过程结束后无需重新制作所要发布的内核。 
SkyEye只是对系统硬件进行了一定程度上的模拟，所以在SkyEye与真实硬件环境相比较而言还是有一定的差距，这对一些与硬件紧密相关的调试可能会有一定的影响，例如驱动程序的调试。不过对于大部分软件的调试，SkyEye已经提供了精度足够的模拟了。 
SkyEye的下一个目标是和eclipse结合，有了图形界面，能为调试和查看源码提供一些方便。 

参考： 
[Linux 系统内核的调试](http://www.ibm.com/developerworks/cn/linux/l-kdb/index.html) 

# 十一  KDB

Linux 内核调试器（KDB）允许您调试 Linux 内核。这个恰如其名的工具实质上是内核代码的补丁，它允许高手访问内核内存和数据结构。KDB 的主要优点之一就是它不需要用另一台机器进行调试：您可以调试正在运行的内核。 
设置一台用于 KDB 的机器需要花费一些工作，因为需要给内核打补丁并进行重新编译。KDB 的用户应当熟悉 Linux 内核的编译（在一定程度上还要熟悉内核内部机理）。 
在本文中，我们将从有关下载 KDB 补丁、打补丁、（重新）编译内核以及启动 KDB 方面的信息着手。然后我们将了解 KDB 命令并研究一些较常用的命令。最后，我们将研究一下有关设置和显示选项方面的一些详细信息。 

## 1  入门

KDB 项目是由 Silicon Graphics 维护的，您需要从它的 FTP 站点下载与内核版本有关的补丁。（在编写本文时）可用的最新 KDB 版本是 4.2。您将需要下载并应用两个补丁。一个是“公共的”补丁，包含了对通用内核代码的更改，另一个是特定于体系结构的补丁。补丁可作为 bz2 文件获取。例如，在运行 2.4.20 内核的 x86 机器上，您会需要 kdb-v4.2-2.4.20-common-1.bz2 和 kdb-v4.2-2.4.20-i386-1.bz2。 
这里所提供的所有示例都是针对 i386 体系结构和 2.4.20 内核的。您将需要根据您的机器和内核版本进行适当的更改。您还需要拥有 root 许可权以执行这些操作。 
将文件复制到 /usr/src/linux 目录中并从用 bzip2 压缩的文件解压缩补丁文件：
```
#bzip2 -d kdb-v4.2-2.4.20-common-1.bz2
#bzip2 -d kdb-v4.2-2.4.20-i386-1.bz2
```
您将获得 kdb-v4.2-2.4.20-common-1 和 kdb-v4.2-2.4-i386-1 文件。 
现在，应用这些补丁：
```
#patch -p1 <kdb-v4.2-2.4.20-common-1
#patch -p1 <kdb-v4.2-2.4.20-i386-1
```
这些补丁应该干净利落地加以应用。查找任何以 .rej 结尾的文件。这个扩展名表明这些是失败的补丁。如果内核树没问题，那么补丁的应用就不会有任何问题。 
接下来，需要构建内核以支持 KDB。第一步是设置 CONFIG_KDB 选项。使用您喜欢的配置机制（xconfig 和 menuconfig 等）来完成这一步。转到结尾处的“Kernel hacking”部分并选择“Built-in Kernel Debugger support”选项。 
您还可以根据自己的偏好选择其它两个选项。选择“Compile the kernel with frame pointers”选项（如果有的话）则设置CONFIG_FRAME_POINTER 标志。这将产生更好的堆栈回溯，因为帧指针寄存器被用作帧指针而不是通用寄存器。您还可以选择“KDB off by default”选项。这将设置 CONFIG_KDB_OFF 标志，并且在缺省情况下将关闭 KDB。我们将在后面一节中对此进行详细介绍。 
保存配置，然后退出。重新编译内核。建议在构建内核之前执行“make clean”。用常用方式安装内核并引导它。 

## 2  初始化并设置环境变量

您可以定义将在 KDB 初始化期间执行的 KDB 命令。需要在纯文本文件 kdb_cmds 中定义这些命令，该文件位于 Linux 源代码树（当然是在打了补丁之后）的 KDB 目录中。该文件还可以用来定义设置显示和打印选项的环境变量。文件开头的注释提供了编辑文件方面的帮助。使用这个文件的缺点是，在您更改了文件之后需要重新构建并重新安装内核。 

## 3  激活 KDB

如果编译期间没有选中 CONFIG_KDB_OFF ，那么在缺省情况下 KDB 是活动的。否则，您需要显式地激活它 － 通过在引导期间将kdb=on 标志传递给内核或者通过在挂装了 /proc 之后执行该工作：
```
#echo "1" >/proc/sys/kernel/kdb
```
倒过来执行上述步骤则会取消激活 KDB。也就是说，如果缺省情况下 KDB 是打开的，那么将 kdb=off 标志传递给内核或者执行下面这个操作将会取消激活 KDB：
```
#echo "0" >/proc/sys/kernel/kdb
```
在引导期间还可以将另一个标志传递给内核。 kdb=early 标志将导致在引导过程的初始阶段就把控制权传递给 KDB。如果您需要在引导过程初始阶段进行调试，那么这将有所帮助。 
调用 KDB 的方式有很多。如果 KDB 处于打开状态，那么只要内核中有紧急情况就自动调用它。按下键盘上的 PAUSE 键将手工调用 KDB。调用 KDB 的另一种方式是通过串行控制台。当然，要做到这一点，需要设置串行控制台并且需要一个从串行控制台进行读取的程序。按键序列 Ctrl-A 将从串行控制台调用 KDB。 

## 4  KDB 命令

KDB 是一个功能非常强大的工具，它允许进行几个操作，比如内存和寄存器修改、应用断点和堆栈跟踪。根据这些，可以将 KDB 命令分成几个类别。下面是有关每一类中最常用命令的详细信息。 

### 4.1  内存显示和修改 
这一类别中最常用的命令是 md 、 mdr 、 mm 和 mmW 。 
md 命令以一个地址／符号和行计数为参数，显示从该地址开始的 line-count 行的内存。如果没有指定 line-count ，那么就使用环境变量所指定的缺省值。如果没有指定地址，那么 md 就从上一次打印的地址继续。地址打印在开头，字符转换打印在结尾。 
mdr 命令带有地址／符号以及字节计数，显示从指定的地址开始的 byte-count 字节数的初始内存内容。它本质上和 md 一样，但是它不显示起始地址并且不在结尾显示字符转换。 mdr 命令较少使用。 
mm 命令修改内存内容。它以地址／符号和新内容作为参数，用 new-contents 替换地址处的内容。 
mmW 命令更改从地址开始的 W 个字节。请注意， mm 更改一个机器字。 
示例 

显示从 0xc000000 开始的 15 行内存：
> [0]kdb> md 0xc000000 15

将内存位置为 0xc000000 上的内容更改为 0x10：
> [0]kdb> mm 0xc000000 0x10

### 4.2  寄存器显示和修改 
这一类别中的命令有 rd 、 rm 和 ef 。 
rd 命令（不带任何参数）显示处理器寄存器的内容。它可以有选择地带三个参数。如果传递了 c 参数，则 rd 显示处理器的控制寄存器；如果带有 d 参数，那么它就显示调试寄存器；如果带有 u 参数，则显示上一次进入内核的当前任务的寄存器组。 
rm 命令修改寄存器的内容。它以寄存器名称和 new-contents 作为参数，用 new-contents 修改寄存器。寄存器名称与特定的体系结构有关。目前，不能修改控制寄存器。 
ef 命令以一个地址作为参数，它显示指定地址处的异常帧。 
示例 
显示通用寄存器组：

>[0]kdb> rd
[0]kdb> rm %ebx 0x25

### 4.3  断点 
常用的断点命令有 bp 、 bc 、 bd 、 be 和 bl 。 
bp 命令以一个地址／符号作为参数，它在地址处应用断点。当遇到该断点时则停止执行并将控制权交予 KDB。该命令有几个有用的变体。 bpa 命令对 SMP 系统中的所有处理器应用断点。 bph 命令强制在支持硬件寄存器的系统上使用它。 bpha 命令类似于 bpa 命令，差别在于它强制使用硬件寄存器。 
bd 命令禁用特殊断点。它接收断点号作为参数。该命令不是从断点表中除去断点，而只是禁用它。断点号从 0 开始，根据可用性顺序分配给断点。 
be 命令启用断点。该命令的参数也是断点号。 
bl 命令列出当前的断点集。它包含了启用的和禁用的断点。 
bc 命令从断点表中除去断点。它以具体的断点号或 * 作为参数，在后一种情况下它将除去所有断点。 

示例 
对函数 sys_write() 设置断点：
> [0]kdb> bp sys_write

列出断点表中的所有断点：
> [0]kdb> bl

清除断点号 1：
> [0]kdb> bc 1

### 4.4  堆栈跟踪 
主要的堆栈跟踪命令有 bt 、 btp 、 btc 和 bta 。 
bt 命令设法提供有关当前线程的堆栈的信息。它可以有选择地将堆栈帧地址作为参数。如果没有提供地址，那么它采用当前寄存器来回溯堆栈。否则，它假定所提供的地址是有效的堆栈帧起始地址并设法进行回溯。如果内核编译期间设置了CONFIG_FRAME_POINTER 选项，那么就用帧指针寄存器来维护堆栈，从而就可以正确地执行堆栈回溯。如果没有设置CONFIG_FRAME_POINTER ，那么 bt 命令可能会产生错误的结果。 
btp 命令将进程标识作为参数，并对这个特定进程进行堆栈回溯。 
btc 命令对每个活动 CPU 上正在运行的进程执行堆栈回溯。它从第一个活动 CPU 开始执行 bt ，然后切换到下一个活动 CPU，以此类推。 
bta 命令对处于某种特定状态的所有进程执行回溯。若不带任何参数，它就对所有进程执行回溯。可以有选择地将各种参数传递给该命令。将根据参数处理处于特定状态的进程。选项以及相应的状态如下： 
D：不可中断状态
R：正运行
S：可中断休眠
T：已跟踪或已停止
Z：僵死
U：不可运行

这类命令中的每一个都会打印出一大堆信息。示例 

跟踪当前活动线程的堆栈：

> [0]kdb> bt

跟踪标识为 575 的进程的堆栈：
> [0]kdb> btp 575

### 4.5  其它命令 
下面是在内核调试过程中非常有用的其它几个 KDB 命令。 
id 命令以一个地址／符号作为参数，它对从该地址开始的指令进行反汇编。环境变量 IDCOUNT 确定要显示多少行输出。 
ss 命令单步执行指令然后将控制返回给 KDB。该指令的一个变体是 ssb ，它执行从当前指令指针地址开始的指令（在屏幕上打印指令），直到它遇到将引起分支转移的指令为止。分支转移指令的典型示例有 call 、 return 和 jump 。 
go 命令让系统继续正常执行。一直执行到遇到断点为止（如果已应用了一个断点的话）。 
reboot 命令立刻重新引导系统。它并没有彻底关闭系统，因此结果是不可预测的。 
ll 命令以地址、偏移量和另一个 KDB 命令作为参数。它对链表中的每个元素反复执行作为参数的这个命令。所执行的命令以列表中当前元素的地址作为参数。 
示例 
反汇编从例程 schedule 开始的指令。所显示的行数取决于环境变量 IDCOUNT ：
> [0]kdb> id schedule

执行指令直到它遇到分支转移条件（在本例中为指令 jne ）为止：
```
[0]kdb> ssb
0xc0105355  default_idle+0x25:  cli
0xc0105356  default_idle+0x26:  mov  0x14(%edx),%eax
0xc0105359  default_idle+0x29:  test %eax, %eax
0xc010535b  default_idle+0x2b:  jne  0xc0105361 default_idle+0x31
```
## 5  技巧和诀窍

调试一个问题涉及到：使用调试器（或任何其它工具）找到问题的根源以及使用源代码来跟踪导致问题的根源。单单使用源代码来确定问题是极其困难的，只有老练的内核黑客才有可能做得到。相反，大多数的新手往往要过多地依靠调试器来修正错误。这种方法可能会产生不正确的问题解决方案。我们担心的是这种方法只会修正表面症状而不能解决真正的问题。此类错误的典型示例是添加错误处理代码以处理 NULL 指针或错误的引用，却没有查出无效引用的真正原因。 
结合研究代码和使用调试工具这两种方法是识别和修正问题的最佳方案。 
调试器的主要用途是找到错误的位置、确认症状（在某些情况下还有起因）、确定变量的值，以及确定程序是如何出现这种情况的（即，建立调用堆栈）。有经验的黑客会知道对于某种特定的问题应使用哪一个调试器，并且能迅速地根据调试获取必要的信息，然后继续分析代码以识别起因。 
因此，这里为您介绍了一些技巧，以便您能使用 KDB 快速地取得上述结果。当然，要记住，调试的速度和精确度来自经验、实践和良好的系统知识（硬件和内核内部机理等）。 

### 5.1  技巧 #1 
在 KDB 中，在提示处输入地址将返回与之最为匹配的符号。这在堆栈分析以及确定全局数据的地址／值和函数地址方面极其有用。同样，输入符号名则返回其虚拟地址。 
示例 

表明函数 sys_read 从地址 0xc013db4c 开始：
```
[0]kdb> 0xc013db4c
0xc013db4c = 0xc013db4c (sys_read)
```
同样，表明 sys_write 位于地址 0xc013dcc8：
```
[0]kdb> sys_write
sys_write = 0xc013dcc8 (sys_write)
```
这些有助于在分析堆栈时找到全局数据和函数地址。 

### 5.2  技巧 #2 
在编译带 KDB 的内核时，只要 CONFIG_FRAME_POINTER 选项出现就使用该选项。为此，需要在配置内核时选择“Kernel hacking”部分下面的“Compile the kernel with frame pointers”选项。这确保了帧指针寄存器将被用作帧指针，从而产生正确的回溯。实际上，您可以手工转储帧指针寄存器的内容并跟踪整个堆栈。例如，在 i386 机器上，%ebp 寄存器可以用来回溯整个堆栈。 
例如，在函数 rmqueue() 上执行第一个指令后，堆栈看上去类似于下面这样：
```
[0]kdb> md %ebp
0xc74c9f38 c74c9f60  c0136c40 000001f0 00000000
0xc74c9f48 08053328 c0425238 c04253a8 00000000
0xc74c9f58 000001f0  00000246 c74c9f6c c0136a25
0xc74c9f68 c74c8000  c74c9f74  c0136d6d c74c9fbc
0xc74c9f78 c014fe45  c74c8000  00000000 08053328
[0]kdb> 0xc0136c40
0xc0136c40 = 0xc0136c40 (__alloc_pages +0x44)
[0]kdb> 0xc0136a25
0xc0136a25 = 0xc0136a25 (_alloc_pages +0x19)
[0]kdb> 0xc0136d6d
0xc0136d6d = 0xc0136d6d (__get_free_pages +0xd)
```
我们可以看到 rmqueue() 被 __alloc_pages 调用，后者接下来又被 _alloc_pages 调用，以此类推。 
每一帧的第一个双字（double word）指向下一帧，这后面紧跟着调用函数的地址。因此，跟踪堆栈就变成一件轻松的工作了。 

### 5.3  技巧 #3 
go 命令可以有选择地以一个地址作为参数。如果您想在某个特定地址处继续执行，则可以提供该地址作为参数。另一个办法是使用rm 命令修改指令指针寄存器，然后只要输入 go 。如果您想跳过似乎会引起问题的某个特定指令或一组指令，这就会很有用。但是，请注意，该指令使用不慎会造成严重的问题，系统可能会严重崩溃。 

### 5.4  技巧 #4 
您可以利用一个名为 defcmd 的有用命令来定义自己的命令集。例如，每当遇到断点时，您可能希望能同时检查某个特殊变量、检查某些寄存器的内容并转储堆栈。通常，您必须要输入一系列命令，以便能同时执行所有这些工作。 defcmd 允许您定义自己的命令，该命令可以包含一个或多个预定义的 KDB 命令。然后只需要用一个命令就可以完成所有这三项工作。其语法如下：
```
[0]kdb> defcmd name "usage" "help"
[0]kdb> [defcmd] type the commands here
[0]kdb> [defcmd] endefcmd
```
例如，可以定义一个（简单的）新命令 hari ，它显示从地址 0xc000000 开始的一行内存、显示寄存器的内容并转储堆栈：
```
[0]kdb> defcmd hari "" "no arguments needed"
[0]kdb> [defcmd] md 0xc000000 1
[0]kdb> [defcmd] rd
[0]kdb> [defcmd] md %ebp 1
[0]kdb> [defcmd] endefcmd
```
该命令的输出会是：
```
[0]kdb> hari
[hari]kdb> md 0xc000000 1
0xc000000 00000001 f000e816 f000e2c3 f000e816
[hari]kdb> rd
eax = 0x00000000 ebx = 0xc0105330 ecx = 0xc0466000 edx = 0xc0466000
....
...
[hari]kdb> md %ebp 1
0xc0467fbc c0467fd0 c01053d2 00000002 000a0200
[0]kdb>
```
### 5.5  技巧 #5 
可以使用 bph 和 bpha 命令（假如体系结构支持使用硬件寄存器）来应用读写断点。这意味着每当从某个特定地址读取数据或将数据写入该地址时，我们都可以对此进行控制。当调试数据／内存毁坏问题时这可能会极其方便，在这种情况中您可以用它来识别毁坏的代码／进程。 
示例 
每当将四个字节写入地址 0xc0204060 时就进入内核调试器：
```
[0]kdb> bph 0xc0204060 dataw 4
```
在读取从 0xc000000 开始的至少两个字节的数据时进入内核调试器：
```
[0]kdb> bph 0xc000000 datar 2
```
## 6  结束语

对于执行内核调试，KDB 是一个方便的且功能强大的工具。它提供了各种选项，并且使我们能够分析内存内容和数据结构。最妙的是，它不需要用另一台机器来执行调试。 

参考： 
[Linux 内核调试器内幕 KDB入门指南 ](http://www.ibm.com/developerworks/cn/linux/l-kdbug/index.html)

# 十二  Kprobes

Kprobes 是 Linux 中的一个简单的轻量级装置，让您可以将断点插入到正在运行的内核之中。 Kprobes 提供了一个强行进入任何内核例程并从中断处理器无干扰地收集信息的接口。使用 Kprobes 可以 轻松地收集处理器寄存器和全局数据结构等调试信息。开发者甚至可以使用 Kprobes 来修改 寄存器值和全局数据结构的值。 
为完成这一任务，Kprobes 向运行的内核中给定地址写入断点指令，插入一个探测器。 执行被探测的指令会导致断点错误。Kprobes 钩住（hook in）断点处理器并收集调试信息。Kprobes 甚至可以单步执行被探测的指令。 

## 1  安装

要安装 Kprobes，需要从 Kprobes 主页下载最新的补丁。 打包的文件名称类似于 kprobes-2.6.8-rc1.tar.gz。解开补丁并将其安装到 Linux 内核： 
```
$tar -xvzf kprobes-2.6.8-rc1.tar.gz 
$cd /usr/src/linux-2.6.8-rc1 
$patch -p1 < ../kprobes-2.6.8-rc1-base.patch
```
Kprobes 利用了 SysRq 键，这个 DOS 时代的产物在 Linux 中有了新的用武之地。您可以在 Scroll Lock键左边找到 SysRq 键；它通常标识为 Print Screen。要为 Kprobes 启用 SysRq 键，需要安装 kprobes-2.6.8-rc1-sysrq.patch 补丁： 
```
$patch -p1 < ../kprobes-2.6.8-rc1-sysrq.patch
```
使用 make xconfig/ make menuconfig/ make oldconfig 配置内核，并 启用 CONFIG_KPROBES 和 CONFIG_MAGIC_SYSRQ标记。 编译并引导到新内核。您现在就已经准备就绪，可以插入 printk 并通过编写简单的 Kprobes 模块来动态而且无干扰地 收集调试信息。 

## 2  编写 Kprobes 模块

对于每一个探测器，您都要分配一个结构体 struct kprobe kp; （参考 include/linux/kprobes.h 以获得关于此数据结构的详细信息）。 

清单 9. 定义 pre、post 和 fault 处理器
``` C
/* pre_handler: this is called just before the probed instruction is
 * executed.
 */
int handler_pre(struct kprobe *p, struct pt_regs *regs) {
printk("pre_handler: p->addr=0x%p, eflags=0x%lx\n",p->addr,
regs->eflags);
return 0;
}
/* post_handler: this is called after the probed instruction is executed
 * (provided no exception is generated).
 */
void handler_post(struct kprobe *p, struct pt_regs *regs, unsigned long flags) {
printk("post_handler: p->addr=0x%p, eflags=0x%lx \n", p->addr,
regs->eflags);
}
/* fault_handler: this is called if an exception is generated for any
 * instruction within the fault-handler, or when Kprobes
 * single-steps the probed instruction.
 */
int handler_fault(struct kprobe *p, struct pt_regs *regs, int trapnr) {
printk("fault_handler:p->addr=0x%p, eflags=0x%lx\n", p->addr,
regs->eflags);
return 0;
}
```
### 2.1  获得内核例程的地址 
在注册过程中，您还需要指定插入探测器的内核例程的地址。使用这些方法中的任意一个来获得内核例程 的地址： 
从 System.map 文件直接得到地址。

例如，要得到 do_fork 的地址，可以在命令行执行 `$grep do_fork /usr/src/linux/System.map` 。

使用 nm 命令。
```
$nm vmlinuz |grep do_fork
```
从 /proc/kallsyms 文件获得地址。
```
$cat /proc/kallsyms |grep do_fork
```
使用 kallsyms_lookup_name() 例程。

这个例程是在 kernel/kallsyms.c 文件中定义的，要使用它，必须启用 CONFIG_KALLSYMS 编译内核。**kallsyms_lookup_name()** 接受一个字符串格式内核例程名， 返回那个内核例程的地址。例如：**kallsyms_lookup_name("do_fork")**;

然后在 init_moudle 中注册您的探测器： 

清单 10. 注册一个探测器
``` C
/* specify pre_handler address
 */
kp.pre_handler=handler_pre;
/* specify post_handler address
 */
kp.post_handler=handler_post;
/* specify fault_handler address
 */
kp.fault_handler=handler_fault;
/* specify the address/offset where you want to insert probe.
 * You can get the address using one of the methods described above.
 */
kp.addr = (kprobe_opcode_t *) kallsyms_lookup_name("do_fork");
/* check if the kallsyms_lookup_name() returned the correct value.
 */
if (kp.add == NULL) {
printk("kallsyms_lookup_name could not find address
for the specified symbol name\n");
return 1;
}
/* or specify address directly.
 * $grep "do_fork" /usr/src/linux/System.map
 * or
 * $cat /proc/kallsyms |grep do_fork
 * or
 * $nm vmlinuz |grep do_fork
 */
kp.addr = (kprobe_opcode_t *) 0xc01441d0;
/* All set to register with Kprobes
 */
       register_kprobe(&kp);
```
一旦注册了探测器，运行任何 shell 命令都会导致一个对 do_fork 的调用，您将可以在控制台上或者运行 dmesg 命令来查看您的 printk。做完后要记得注销探测器： 
unregister_kprobe(&kp); 
下面的输出显示了 kprobe 的地址以及 eflags 寄存器的内容： 
```
$tail -5 /var/log/messages 
Jun 14 18:21:18 llm05 kernel: pre_handler: p->addr=0xc01441d0, eflags=0x202 
Jun 14 18:21:18 llm05 kernel: post_handler: p->addr=0xc01441d0, eflags=0x196 
```
2.2  获得偏移量 
您可以在例程的开头或者函数中的任意偏移位置插入 printk（偏移量必须在指令范围之内）。 下面的代码示例展示了如何来计算偏移量。首先，从对象文件中反汇编机器指令，并将它们 保存为一个文件： 
> $objdump -D /usr/src/linux/kernel/fork.o > fork.dis

其结果是： 

清单 11. 反汇编的 fork
```
000022b0 <do_fork>:
   22b0:       55                      push   %ebp
   22b1:       89 e5                   mov    %esp,%ebp
   22b3:       57                      push   %edi
   22b4:       89 c7                   mov    %eax,%edi
   22b6:       56                      push   %esi
   22b7:       89 d6                   mov    %edx,%esi
   22b9:       53                      push   %ebx
   22ba:       83 ec 38                sub    $0x38,%esp
   22bd:       c7 45 d0 00 00 00 00    movl   $0x0,0xffffffd0(%ebp)
   22c4:       89 cb                   mov    %ecx,%ebx
   22c6:       89 44 24 04             mov    %eax,0x4(%esp)
   22ca:       c7 04 24 0a 00 00 00    movl   $0xa,(%esp)
   22d1:       e8 fc ff ff ff          call   22d2 <do_fork+0x22>
   22d6:       b8 00 e0 ff ff          mov    $0xffffe000,%eax
   22db:       21 e0                   and    %esp,%eax
   22dd:       8b 00                   mov    (%eax),%eax
```
要在偏移位置 0x22c4 插入探测器，先要得到与例程的开始处相对的偏移量 0x22c4 - 0x22b0 = 0x14 ，然后将这个偏移量添加到 do_fork 的地址 0xc01441d0 + 0x14 。（运行 $cat /proc/kallsyms | grep do_fork 命令以获得 do_fork 的地址。） 
您还可以将 do_fork 的相对偏移量 0x22c4 - 0x22b0 = 0x14 添加到 kallsyms_lookup_name("do_fork"); 的输入，即：0x14 + kallsyms_lookup_name("do_fork"); 

2.3  转储内核数据结构 
现在，让我们使用修改过的用来转储数据结构的 Kprobe post_handler 来转储运行在系统上的所有作业的一些组成部分： 

清单 12. 用来转储数据结构的修改过的 Kprope post_handler
``` C
void handler_post(struct kprobe *p, struct pt_regs *regs, unsigned long flags){
    struct task_struct *task;
    read_lock(&tasklist_lock);
    for_each_process(task) {
    printk("pid =%x task-info_ptr=%lx\n", task->pid,
    task->thread_info);
    printk("thread-info element status=%lx,flags=%lx, cpu=%lx\n",
    task->thread_info->status, task->thread_info->flags,
    task->thread_info->cpu);
}
    read_unlock(&tasklist_lock);
}
```
这个模块应该插入到 do_fork 的偏移位置。 

清单 13. pid 1508 和 1509 的结构体 thread_info 的输出
```
$tail -10 /var/log/messages
Jun 22 18:14:25 llm05 kernel: thread-info element status=0,flags=0, cpu=1
Jun 22 18:14:25 llm05 kernel: pid =5e4 task-info_ptr=f5948000
Jun 22 18:14:25 llm05 kernel: thread-info element status=0,flags=8, cpu=0
Jun 22 18:14:25 llm05 kernel: pid =5e5 task-info_ptr=f5eca000
```
### 2.4  启用奇妙的 SysRq 键 
为了支持 SysRq 键，我们已经进行了编译。这样来启用它： 
> $echo 1 > /proc/sys/kernel/sysrq

现在，您可以使用 Alt+SysRq+W 在控制台上或者到 /var/log/messages 中去查看所有插入的内核探测器。 

清单 14. /var/log/messages 显示出在 do_fork 插入了一个 Kprobe
```
Jun 23 10:24:48 linux-udp4749545uds kernel: SysRq : Show kprobes
Jun 23 10:24:48 linux-udp4749545uds kernel:
Jun 23 10:24:48 linux-udp4749545uds kernel: [<c011ea60>] do_fork+0x0/0x1de
```
## 3  使用 Kprobes 更好地进行调试

由于探测器事件处理器是作为系统断点中断处理器的扩展来运行，所以它们很少或者根本不依赖于系统 工具 —— 这样可以被植入到大部分不友好的环境中（从中断时间和任务时间到禁用的上下文间切换和支持 SMP 的代码路径）—— 都不会对系统性能带来负面影响。 
使用 Kprobes 的好处有很多。不需要重新编译和重新引导内核就可以插入 printk。为了进行调试可以记录 处理器寄存器的日志，甚至进行修改 —— 不会干扰系统。类似地，同样可以无干扰地记录 Linux 内核数据结构的日志，甚至 进行修改。您甚至可以使用 Kprobes 调试 SMP 系统上的竞态条件 —— 避免了您自己重新编译和重新引导的所有 麻烦。您将发现内核调试比以往更为快速和简单。 

参考： 
[使用 Kprobes 调试内核](http://www.ibm.com/developerworks/cn/linux/l-kprobes.html) 

本文参考：
1.  Linux内核设计与实现   P243 第十八章 调试 
2.  [linux内核调试指南](http://www.360doc.com/content/10/1215/15/1378815_78374144.shtml)
