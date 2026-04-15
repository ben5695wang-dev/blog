---
title: "Linux诊断和故障排除系列(十一) -- dump设置和分析"
description: 
date: 2024-07-11T20:02:29+08:00
image: 
math: 
license: 
comments: true
draft: false
toc: true
# 分类（Categories） - 适合大类、主要主题
categories:
  - 技术
  - Linux
  - Linux诊断和故障排除
# 标签（Tags） - 适合具体关键词、多个细标签
tags:
  - Linux
  - Linux诊断和故障排除
  - dump设置和分析
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 安装并启动kdump
```shell
$ dnf install kexec-tools       //安装kdump
$ systemctl enable --now kdump      //启动kdump
$ systemctl status kdump				<<<<<<查看kdump服务是否启动，这个服务是做dump的
● kdump.service - Crash recovery kernel arming
   Loaded: loaded (/usr/lib/systemd/system/kdump.service; enabled; vendor preset: enabled)
   Active: active (exited) since Sun 2024-04-28 15:35:41 CST; 1 months 20 days ago
 Main PID: 678 (code=exited, status=0/SUCCESS)
   CGroup: /system.slice/kdump.service

Apr 28 15:35:38 hecs-295729 systemd[1]: Starting Crash recovery kernel arming...
Apr 28 15:35:41 hecs-295729 kdumpctl[678]: kexec: loaded kdump kernel
Apr 28 15:35:41 hecs-295729 kdumpctl[678]: Starting kdump: [OK]
Apr 28 15:35:41 hecs-295729 systemd[1]: Started Crash recovery kernel arming.
```
# 2. 配置kdump
## 2.1 配置kdump位置
默认行为是将其存储在本地文件系统的 /var/crash/ 目录中，可以在/etc/kdump.conf 里面配置。
崩溃转储通常以一个文件形式存储在本地文件系统中，直接写入设备。或者，您可以为崩溃转储进行设置，以通过使用 NFS 或 SSH 协议的网络进行发送。一次只能设置其中一个选项来保留崩溃转储文件。

要将崩溃转储文件保存在本地文件系统的 /var/crash/ 目录中，请编辑 /etc/kdump.conf 文件并指定路径：
path /var/crash
选项 path /var/crash 代表 kdump 在其中保存崩溃转储文件的文件系统的路径。

注意:
当您在 /etc/kdump.conf 文件中指定转储目标时，路径是相对于指定的转储目标。
当您没有在 /etc/kdump.conf 文件中指定转储目标时，该路径表示根目录的绝对路径。
配置例子： kdump 目标配置
```shell
$ grep -v ^# /etc/kdump.conf | grep -v ^$
ext4 /dev/mapper/vg00-varcrashvol
path /var/crash
core_collector makedumpfile -c --message-level 1 -d 31
```
此处，转储目标被指定(ext4 /dev/mapper/vg00-varcrashvol)，因此在 /var/crash 处挂载。path 选项也被设置为 /var/crash，因此 kdump 会将 vmcore 文件保存在 /var/crash/var/crash 目录中。
重启 kdump 服务以使更改生效：
```shell
sudo systemctl restart kdump.service
```
## 2.2 配置kdump内存相关
### 2.2.1 估算 kdump 大小
```shell
 makedumpfile --mem-usage /proc/kcore 		//查看dump的预估大小，如果磁盘空间不够，那么dump也会失败

TYPE            PAGES                   EXCLUDABLE      DESCRIPTION
----------------------------------------------------------------------
ZERO            19894                   yes             Pages filled with zero
NON_PRI_CACHE   123490                  yes             Cache pages without private flag
PRI_CACHE       533911                  yes             Cache pages with private flag
USER            55632                   yes             User process pages
FREE            197063                  yes             Free pages
KERN_DATA       82981                   no              Dumpable kernel data 

page size:              4096            
Total pages on system:  1012971         
Total size on system:   4149129216       Byte			//预估4G左右
//makedumpfile --mem-usage 命令会以页为单位报告所需的内存。这意味着您必须根据内核页面大小计算所使用的内存大小。
//默认情况下，RHEL 内核在 AMD64 和 Intel 64 CPU 构架上使用 4 KB 大小的页，在 IBM POWER 构架上使用 64 KB 大小的页。
```
### 2.2.2 配置 kdump 内存用量
kdump 的自动内存分配因系统硬件架构和可用内存大小而异。
例如，在 AMD64 和 Intel 64 上，crashkernel=auto 参数仅在可用内存超过 1GB 时才起作用。64 位 ARM 架构和 IBM Power 系统需要超过 2 GB 的可用内存。
a. 准备 crashkernel= 选项。
要保留 128 MB 内存，请使用： crashkernel=128M
```shell
$ grep crashkernel /proc/cmdline    //查看启动参数里面的carshkernel设置，一般设置为crashkernel=auto，不然可能因为内存不足而无法生成dump
BOOT_IMAGE=/boot/vmlinuz-3.10.0-1160.92.1.el7.x86_64 root=UUID=7c5d72cf-4d6b-4cd2-ac0d-fcc9270be4c4 ro net.ifnames=0 consoleblank=600 console=tty0 console=ttyS0,115200n8 spectre_v2=off nopti crashkernel=auto spectre_v2=retpoline LANG=en_US.UTF-8
// crashkernel可以有以下几个选项
// 固定大小，crashkernel=256M 
// 比例，crashkernel=1/4 预留系统总内存的四分之一或一半。
// 自动：crashkernel=auto 让系统根据总内存的大小自动决定预留的内存量。例如，在 Red Hat Enterprise Linux 中，如果系统内存小于 2GB，可能不会预留内存；如果内存在 2GB 到 4GB 之间，可能会预留 128MB；如果内存超过 4GB，可能会预留更多的内存。
// 大小加偏移，crashkernel=256M@64M。这里 256M 是预留的内存大小，64M 是从起始内存地址开始的偏移量。
// 禁用：如果不想使用 Kdump，可以设置 crashkernel=0 或者不包含 crashkernel 参数
```
b. 将 crashkernel= 选项应用到引导装载程序配置：
```shell
$ grubby --update-kernel=ALL --args="crashkernel=<value>"
```
将 替换为您在上一步中准备的 crashkernel= 选项的值。
## 2.3 配置 kdump 核心收集器
kdump 服务使用 core_collector 程序捕获崩溃转储镜像。在 RHEL 中，makedumpfile 工具是默认的内核收集器。它通过以下方式帮助缩小转储文件：
压缩崩溃转储文件的大小，并使用不同的转储级别只复制所需的页。
排除不必要的崩溃转储页。
过滤要包含在崩溃转储中的页类型。
```shell
$ cat /etc/kdump.conf | grep -v ^$ | grep -v ^#			//kdump的配置文件
path /var/crash			//这里有dump文件的位置
core_collector makedumpfile -l --message-level 1 -d 31			//core_collector 是一个在 Linux 系统中用于收集和压缩内核崩溃时的内存转储信息的工具
//  -l：这个选项告诉 makedumpfile 以列表模式运行，即它将打印出内存转储的每个块的信息，但不会实际写入文件。
//  --message-level 1：这个选项设置消息的详细级别。数字 1 表示只打印错误消息。
//  -d 31：这个选项指定了调试级别，31 是一个特定的位掩码，用于控制输出哪些调试信息
```
## 2.4 配置 kdump 默认失败响应 
默认情况下，当 kdump 不能在配置的目标位置创建崩溃转储文件时，系统会重启，转储在此过程中会丢失。您可以更改默认故障响应，并配置 kdump 以执行不同的操作，以防无法将内核转储保存到主目标。额外的操作是：
dump_to_rootfs 将内核转储保存到 root 文件系统。
reboot 重启系统，在此过程中会丢失内核转储。
halt 停止系统，在此过程中会丢失内核转储。
poweroff 关闭系统，在此过程中会丢失内核转储。
shell 从 initramfs 中运行 shell 会话，您可以手动记录内核转储。
final_action 在 kdump 成功或在 shell 或 dump_to_rootfs 失败操作完成后，启用额外的操作，如 reboot、halt 和 poweroff。默认为 reboot。
failure_action 指定在内核崩溃中转储可能失败时要执行的操作。默认为 reboot

### 2.4.1 配置 kdump 默认失败响应 流程
以 root 用户身份，从 /etc/kdump.conf 配置文件中 #failure_action 行的开头删除哈希符号(#)。
将值替换为所需操作。
failure_action poweroff
## 2.5 支持的 kdump 过滤等级
要缩小转储文件的大小，kdump 使用 makedumpfile 内核收集器压缩数据，并排除不需要的信息，例如，您可以使用 -8 级别来删除 hugepages 和 hugetlbfs 页。makedumpfile 当前支持的级别可在 Filtering levels for kdump 表中看到。
kdump的过滤级别
选项 描述
1 零页
2 缓存页
4 缓存私有
8 用户页
16 可用页
# 3. 测试 kdump 配置
不要执行以下操作，以下操作会导致Linux重启！！！
```shell
$ kdumpctl restart      //重启服务
$ kdumpctl status       //检查服务状态
$ echo c > /proc/sysrq-trigger	//强制生成一个dump，这个命令会导致操作系统会重启！！！在内核重启时，address-YYYY-MM-DD-HH:MM:SS/vmcore 文件在您在 /etc/kdump.conf 文件中指定的位置创建。默认值为 /var/crash/。里面有一个vmcore，就是dump文件。
```
系统崩溃后，kdump 服务在转储文件(vmcore)中捕获内核内存，它还生成额外的诊断文件，以帮助故障排除和事后分析。
## 3.1 kdump 生成的文件：
vmcore - 包含崩溃时系统内存的主内核内存转储文件。它包含根据 kdump 配置中指定的 core_collector 程序配置的数据。默认情况下，内核数据结构、处理信息、堆栈跟踪和其他诊断信息。
vmcore-dmesg.txt - panic 的主内核中的内核环缓冲区日志的内容(dmesg)。
kexec-dmesg.log - 包含收集 vmcore 数据的二级 kexec 内核执行中的内核和系统日志消息。


