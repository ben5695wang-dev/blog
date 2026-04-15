---
title: "Linux诊断和故障排除系列(十) -- 硬件问题日志"
description: 
date: 2024-07-10T20:02:29+08:00
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
  - 硬件问题日志
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 显示硬件相关信息
通过ls命令显示信息
lshw（List Hardware）	列出所有硬件信息，包括CPU、内存、声卡、显卡等。
lscpu		显示CPU信息
lsmem       显示内存信息
lsblk		显示Disk信息
lsraid：显示RAID设备信息。
lsscsi	显示Disk(SCSI)信息
lspci		显示PCI devices信息
lsusb		显示USB devices信息

//以下命令未验证
lsdev：显示系统设备列表。
lsdma：显示DMA（Direct Memory Access）通道的使用情况。
lsgpio：显示GPIO（通用输入输出）引脚的状态（在支持的系统上）。
lsnd：显示声音设备信息。
lsnet：显示网络设备信息。
lsbatt：显示电池状态信息（如果系统支持）。
lsirq：显示中断请求（IRQ）的使用情况。

```shell
# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                2
On-line CPU(s) list:   0,1
Thread(s) per core:    2
Core(s) per socket:    1
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 85
Model name:            Intel(R) Xeon(R) Gold 6161 CPU @ 2.20GHz
Stepping:              4
CPU MHz:               2200.000
BogoMIPS:              4400.00
Hypervisor vendor:     KVM
Virtualization type:   full
L1d cache:             32K
L1i cache:             32K
L2 cache:              1024K
L3 cache:              30976K
NUMA node0 CPU(s):     0,1
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology nonstop_tsc eagerfpu pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single ssbd rsb_ctxsw ibrs ibpb stibp fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx avx512f avx512dq rdseed adx smap clflushopt clwb avx512cd avx512bw avx512vl xsaveopt xsavec xgetbv1 arat md_clear spec_ctrl intel_stibp flush_l1d
```
显示BIOS数据
dmidecode 显示BIOS数据，dump stats
dmidecode -t memory 显示BIOS数据里的内存信息，Memory
# 2. 硬件日志
dmesg //硬件相关的log一般会发到dmesg里
mcelog //(Machine check exception log)mcelog会跟踪硬件信息，并报告给journald
journalctl -u mcelog.service
## 2.1 mcelog
```shell
$ yum install mcelog                        //安装mcelog
$ systemctl enable --now mcelog             //启动mcelog
$ systemctl status mcelog
$ journalctl -u mcelog.service              //通过journalctl来查看mcelog的日志
$ dmesg | grep -i mce						//通过dmesg来查看mcelog的日志
$ mcelog			//通过mcelog命令来查看mcelog的日志
$ mcelog --from "2024-07-11 00:00:00"		//使用 --from 选项来查看特定时间之后的错误：
```
# 3. Memory Test
memtest86+ //在启动的时候有一个选项，可以运行内存测试
a. 安装memtest86+
```shell
yum install memtest86+
```
b. 运行memtest-setup
```shell
$ memtest-setup			//运行就行，什么都不用做
GRUB 2 detected, installing template...
GRUB 2 template installed.
Do not forget to regenerate your grub.cfg by:
  # grub2-mkconfig -o /boot/grub2/grub.cfg
Setup complete.
```
c. 更新GRUB2配置，以使memtest作为一个启动选项
```shell
$ grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found memtest image: /boot/elf-memtest86+-5.01
done
Setup complete.
$ reboot
```
d. 在启动项里，最下面多了一个 Red Hat Enterprise Linux Memtest memtest86+-5.01 选项
