---
title: 基于linux 3.10的yaffs2移植
categories: Linux
tags:
  - linux移植
date: 2013-11-02 13:07:41
---

最近想起来还有一块mini2440的开发板很久没有使用了，所以想移植一个基于linux3.10的linux系统，但是在移植yaffs2文件系统的时候出现了一些问题，我将其记录下来给其他同学解决同样的问题提供帮助。

1. 首先通过git下载yaffs2代码。然后进入yaffs2文件夹中执行patch-ker.sh，给linux源代码打上patch。
```
    $ git clone git://www.aleph1.co.uk/yaffs2
    $ cd yaffs2/
    $ ./patch-ker.sh c m ../linux3.10-mini2440 
```
2. 然后在linux的源代码fs中多了一个yaffs2的文件夹，到此yaffs2文件系统就已经添加到linux3.10中了。在Linux内核源代码根目录运行：make menuconfig，移动上下按键进行配置：
```
    `File Systems
    	---> Miscellaneous filesystems
    		---> [*]YAFFS2 file system support
```
并按空格选中它，这样我们就在内核中添加了yaffs2文件系统的支持，按“Exit”退出内核配置。 
3. 编译linux源代码。
```
    $ make zImage
    scripts/kconfig/conf --silentoldconfig Kconfig
      CHK     include/generated/uapi/linux/version.h
      CHK     include/generated/utsrelease.h
    make[1]: `include/generated/mach-types.h' is up to date.
      CALL    scripts/checksyscalls.sh
      CC      scripts/mod/devicetable-offsets.s
      GEN     scripts/mod/devicetable-offsets.h
      HOSTCC  scripts/mod/file2alias.o
      HOSTLD  scripts/mod/modpost
      CHK     include/generated/compile.h
      CC      fs/yaffs2/yaffs_ecc.o
      CC      fs/yaffs2/yaffs_vfs.o
    fs/yaffs2/yaffs_vfs.c: In function 'yaffs_proc_debug_write':
    fs/yaffs2/yaffs_vfs.c:3304: warning: comparison of distinct pointer types lacks a cast
    fs/yaffs2/yaffs_vfs.c: In function 'init_yaffs_fs':
    fs/yaffs2/yaffs_vfs.c:3398: error: implicit declaration of function 'create_proc_entry'
    fs/yaffs2/yaffs_vfs.c:3399: warning: assignment makes pointer from integer without a cast
    fs/yaffs2/yaffs_vfs.c:3402: error: dereferencing pointer to incomplete type
    fs/yaffs2/yaffs_vfs.c:3403: error: dereferencing pointer to incomplete type
    fs/yaffs2/yaffs_vfs.c:3404: error: dereferencing pointer to incomplete type
    make[2]: *** [fs/yaffs2/yaffs_vfs.o] Error 1
    make[1]: *** [fs/yaffs2] Error 2
    make: *** [fs] Error 2`</pre> 
```
编译fs/yaffs2/yaffs_vfs.c时出现错误，function 'create_proc_entry'没有申明。Google之后才知道原来这个接口在linux-3.10被删除了，应该使用proc_create代替。

参考：[What's coming in 3.10, part 2](https://lwn.net/Articles/549737/)

4. 修改fs/yaffs2/yaffs_vfs.c
``` C
    @@ -3384,12 +3384,6 @@ static struct file_system_to_install fs_to_install[] = {
            {NULL, 0}
     };

    +static const struct file_operations yaffs_fops = {
    +        .owner = THIS_MODULE,
    +        .read = yaffs_proc_read,
    +        .write = yaffs_proc_write,
    +};
    +
     static int __init init_yaffs_fs(void)
     {
            int error = 0;
    @@ -3401,9 +3395,9 @@ static int __init init_yaffs_fs(void)
            mutex_init(&yaffs_context_lock);

            /* Install the proc_fs entries */
    +       my_proc_entry = proc_create("yaffs",
    +                                         S_IRUGO | S_IFREG, YPROC_ROOT, &yaffs_fops);
    +#if 0
    -       my_proc_entry = create_proc_entry("yaffs",
    -                                         S_IRUGO | S_IFREG, YPROC_ROOT);
    -
            if (my_proc_entry) {
                    my_proc_entry->write_proc = yaffs_proc_write;
                    my_proc_entry->read_proc = yaffs_proc_read;
    @@ -3411,7 +3405,7 @@ static int __init init_yaffs_fs(void)
            } else {
                    return -ENOMEM;
             }
    +#endif
    -
            /* Now add the file system entries */

            fsinst = fs_to_install; 
```
5. 修改之后保存，然后再编译就可以成功了。
```
    $ make zImage
      CHK     include/generated/uapi/linux/version.h
      CHK     include/generated/utsrelease.h
    make[1]: `include/generated/mach-types.h' is up to date.
      CALL    scripts/checksyscalls.sh
      CC      scripts/mod/devicetable-offsets.s
      GEN     scripts/mod/devicetable-offsets.h
      HOSTCC  scripts/mod/file2alias.o
      HOSTLD  scripts/mod/modpost
      CHK     include/generated/compile.h
      CC      fs/yaffs2/yaffs_vfs.o
    fs/yaffs2/yaffs_vfs.c: In function 'yaffs_proc_debug_write':
    fs/yaffs2/yaffs_vfs.c:3304: warning: comparison of distinct pointer types lacks a cast
    fs/yaffs2/yaffs_vfs.c: At top level:
    fs/yaffs2/yaffs_vfs.c:3389: warning: initialization from incompatible pointer type
    fs/yaffs2/yaffs_vfs.c:3390: warning: initialization from incompatible pointer type
      CC      fs/yaffs2/yaffs_guts.o
      CC      fs/yaffs2/yaffs_checkptrw.o
      CC      fs/yaffs2/yaffs_packedtags1.o
      CC      fs/yaffs2/yaffs_packedtags2.o
      CC      fs/yaffs2/yaffs_nand.o
      CC      fs/yaffs2/yaffs_tagscompat.o
      CC      fs/yaffs2/yaffs_tagsmarshall.o
      CC      fs/yaffs2/yaffs_mtdif.o
      CC      fs/yaffs2/yaffs_nameval.o
      CC      fs/yaffs2/yaffs_attribs.o
      CC      fs/yaffs2/yaffs_allocator.o
      CC      fs/yaffs2/yaffs_yaffs1.o
      CC      fs/yaffs2/yaffs_yaffs2.o
      CC      fs/yaffs2/yaffs_bitmap.o
      CC      fs/yaffs2/yaffs_summary.o
      CC      fs/yaffs2/yaffs_verify.o
      LD      fs/yaffs2/yaffs.o
      LD      fs/yaffs2/built-in.o
      LD      fs/built-in.o
      LINK    vmlinux
      LD      vmlinux.o
      MODPOST vmlinux.o
      GEN     .version
      CHK     include/generated/compile.h
      UPD     include/generated/compile.h
      CC      init/version.o
      LD      init/built-in.o
      KSYM    .tmp_kallsyms1.o
      KSYM    .tmp_kallsyms2.o
      LD      vmlinux
      SORTEX  vmlinux
      SYSMAP  System.map
      OBJCOPY arch/arm/boot/Image
      Kernel: arch/arm/boot/Image is ready
      GZIP    arch/arm/boot/compressed/piggy.gzip
      AS      arch/arm/boot/compressed/piggy.gzip.o
      LD      arch/arm/boot/compressed/vmlinux
      OBJCOPY arch/arm/boot/zImage
      Kernel: arch/arm/boot/zImage is ready
```