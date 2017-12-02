---
title: ACPI DEBUG方法
categories: 电源管理
tags:
  - 功耗
  - ACPI
date: 2013-01-12 23:56:24
---

# 获取ACPI TABLE的方法

## 1. 首先要安装acpidump和iasl命令
  有的机器上需要安装pmtools才可以。 
``` shell
$ sudo apt-get install acpidump 
$ sudo apt-get install iasl 
$ sudo apt-get install pmtools 
```
## 2. 导出ACPI的数据。 
``` shell
$ sudo cat /proc/acpi/dsdt > dstd.dat
```
  或者
``` shell
$ sudo cat /sys/firmware/acpi/tables/DSDT > /tmp/dsdt.dat
```
  如果找不到该文件可以使用下面的命令：
``` shell
$ sudo acpidump > acpi.dat
```
## 3. 分离各表格数据，会生成多个数据文件。
``` shell
 $ acpixtract -a acpi.dat 
 Acpi table [DSDT] - 41555 bytes written to DSDT.dat 
 Acpi table [FACS] - 64 bytes written to FACS.dat 
 Acpi table [FACP] - 116 bytes written to FACP.dat 
 Acpi table [APIC] - 132 bytes written to APIC.dat 
 Acpi table [ASF!] - 99 bytes written to ASF!.dat 
 Acpi table [MCFG] - 60 bytes written to MCFG.dat 
 Acpi table [TCPA] - 50 bytes written to TCPA.dat 
 Acpi table [SLIC] - 374 bytes written to SLIC.dat 
 Acpi table [HPET] - 56 bytes written to HPET.dat 
 Acpi table [DMAR] - 456 bytes written to DMAR.dat 
 Acpi table [RSDT] - 68 bytes written to RSDT.dat 
 Acpi table [RSDP] - 20 bytes written to RSDP.dat 
```
## 4. 反汇编DSDT.dat
``` shell
 $ iasl -d DSDT.dat 
 Intel ACPI Component Architecture 
 AML Disassembler version 20100528 [Oct 15 2010] 
 Copyright (c) 2000 - 2010 Intel Corporation 
 Supports ACPI Specification Revision 4.0a 
 Loading Acpi table from file DSDT.dat 
 Acpi table [DSDT] successfully installed and loaded 
 Pass 1 parse of [DSDT] 
 Pass 2 parse of [DSDT] 
 Parsing Deferred Opcodes (Methods/Buffers/Packages/Regions) 
 ...........................................................
 ...........................................................
 ...........................................................
 ...........................................................
 ...........................................................
 ...........................................................
 ...........................................................
 ...................................................... 
 Parsing completed 
 Disassembly completed, written to "DSDT.dsl" 
```
## 5. 重新编译DSL
根据需要修改**DSDT.dsl**文件，然后重新编译该文件，会生成**DSDT.aml**和**DSDT.hex**两个文件。
``` shell
 $ iasl -tc dsdt.dsl 
 Intel ACPI Component Architecture 
 ASL Optimizing Compiler version 20100528 [Oct 15 2010] 
 Copyright (c) 2000 - 2010 Intel Corporation 
 Supports ACPI Specification Revision 4.0a 
 ...................................................... 
 ASL Input: DSDT.dsl - 8762 lines, 320704 bytes, 3778 keywords 
 AML Output: DSDT.aml - 39328 bytes, 780 named objects, 2998 executable opcodes 
 Compilation complete. 0 Errors, 41 Warnings, 0 Remarks, 988 Optimizations 
```
## 6. 更新DSDT的方法。 
  - 其一(没有测试过，不知道能否成功): 
  使用新的dsdt.aml代替BIOS中的版本: 
``` shell
  cp /boot/initrd.img-2.6.20-16-generic /boot/initrd.img-2.6.20-16-generic-bak 
  sudo cp dsdt.aml /etc/initramfs-tools/DSDT.aml 
  sudo update-initramfs -u -k all 
```
  - 其二（需要安装acpiexec，为找到）
 ``` shell
 iasl -sa DSDT.dsl # e.g. recompile a modified, disassembled DSDT for 
  initramfs inclusion (see later) 
 acpiexec DSDT.dat # parse, interpret and load a DSDT in userspace 
 ```
  - 其三： 这个方法需要重新编译内核。 
  首先将DSDT.hex文件复制到内核源码的include目录下。 
 ``` shell
 $ cp DSDT.hex $SRC/include/
 ```
  然后修改内核配置文件.config 
``` shell
  CONFIG_STANDALONE=n 
  CONFIG_ACPI_CUSTOM_DSDT=y 
  CONFIG_ACPI_CUSTOM_DSDT_FILE="DSDT.hex" 
```
  重新编译，更新内核，然后启动，可以在dmesg中看到
``` shell
 [ 0.000000] ACPI: Override [DSDT- CDV-TPT], this is unsafe: tainting kernel 
 [ 0.000000] Disabling lock debugging due to kernel taint 
 [ 0.000000] ACPI: DSDT @ 0x7f378010 Table override, replaced with: 
 [ 0.000000] ACPI: DSDT c0f0f140 0703F (v02 Intel CDV-TPT 00000000 INTL 20091214)
 ```
  就说明DSDT已经被替换成修改过的了。

## 7. 打开CONFIG_ACPI_DEBUG
在DSDT中添加LOG。 首先应该打开`CONFIG_ACPI_DEBUG=y`。
``` shell
 acpi.debug_level= ACPI_LV_DEBUG_OBJECT， 
 acpi.debug_layer= ACPI_TABLES | ACPI_NAMESPACE，
``` 
  使用将想要的变量或者字符串赋值给Debug即可。 
> (Store (AAAA, Debug) 