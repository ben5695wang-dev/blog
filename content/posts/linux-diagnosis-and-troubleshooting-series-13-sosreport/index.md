---
title: "Linux诊断和故障排除系列(十三) -- 官方支持数据sos_report及其分析可视化软件"
description: 
date: 2024-07-13T20:02:29+08:00
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
  - 官方支持数据sos_report及其分析可视化软件
build:
    list: always    # Change to "never" to hide the page from the list
---

sos_report 是一个在多种Linux发行版(Red Hat/Centos/Ubuntu/Debian/Fedora等及其衍生版)中默认自带的工具，用于收集系统配置和诊断信息的工具。它帮助用户收集系统的状态信息，如正在运行的内核版本、加载的模块和系统和服务配置文件之类的信息。
Sosreport在你需要获得redhat等官方的技术支持时需要它，Redhat等官方工程师会要求你服务器上的sosreport，从分析 sos report 命令输出的数据开始，以帮助故障排除。
https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/8/html/generating_sos_reports_for_technical_support/generating-an-sos-report-for-technical-support_generating-sos-reports-for-technical-support

在RHEL 4.5之前，可能使用 sysreport 命令，
在RHEL 8及以后，sos_report命令更名为 sos report


报告中收集的信息包括来自 RHEL 系统的配置详情、系统信息和诊断信息，例如：
运行的内核版本。
载入的内核模块。
系统和服务配置文件。
诊断命令输出。
安装的软件包列表。
sos 程序将其收集的数据写入名为 sosreport- <host_name> - <support_case_number> - <YYYY-MM-DD> - &lt;unique_random_characters>.tar.xz 的归档。

RHEL 8+ 中的 sos report 等同于 RHEL7 及更早版本中的 sosreport 命令。因此，sos report 与 sosreport 本质上是相同的，只是在 sos 软件包中使用的不同命令语法。

在 RHEL 7 及更早的版本中，安装的软件包名称是 sos，从系统中获取数据的命令是 sosreport。
从 RHEL 8+ 开始，命令名称被改为只需要 sos 及可选参数 report 来执行与旧的 sos 软件包中 sosreport 同样的数据收集活动。

# 2. 安装 sos 软件包
您需要 root 权限。安装 sos 软件包。
```shell
[root@server ~]# yum install sos
```
验证步骤，使用 rpm 实用程序验证是否安装了 sos 软件包。
```shell
[root@server ~]# rpm -q sos
sos-4.2-15.el8.noarch
```
# 3. 从命令行生成 sos 报告
## 3.1 先决条件：
已安装 sos 软件包。
您需要 root 权限。
## 3.2 生成sos报告
运行 sos report 命令并按照屏幕的说明进行操作
```shell
$ sudo sos report
```
记录下控制台输出末尾显示的 sos 报告文件名称。
```shell
...
Finished running plugins
Creating compressed archive...

Your sos report has been generated and saved in:
/var/tmp/sosreport-server1-12345678-2022-04-17-qmtnqng.tar.xz

Size    16.51MiB
Owner   root
sha256  bf303917b689b13f0c059116d9ca55e341d5fadcd3f1473bef7299c4ad2a7f4f

Please send this file to your support representative.
```
您可以使用 --batch 选项在不提示输入交互式输入的情况下生成 sos 报告。
```shell
[user@server1 ~]$ sudo sos report --batch --case-id <8-digit_case_number>
```
您还可以使用 --clean 选项来处理一个刚好的 sos 报告。
```shell
[user@server1 ~]$ sudo sos report --clean
```
验证步骤：
验证 sos 实用程序在 /var/tmp/ 中创建了与命令输出的描述匹配的存档。
```shell
[user@server1 ~]$ sudo ls -l /var/tmp/sosreport*
[sudo] password for user:
-rw-------. 1 root root 17310544 Sep 17 19:11 /var/tmp/sosreport-server1-12345678-2022-04-17-qmtnqng.tar.xz
```
## 3.3 sos 报告处理敏感信息
sos 实用程序提供了一个功能来处理潜在的敏感数据，如用户名、主机名、IP 或 MAC 地址或其他用户指定的关键字。原始 sos 报告里的数据是保持不变，新的 *-obfuscated.tar.xz 文件被生成并可以与第三方共享。
针对 sos report运行 sos clean 命令，并按照屏幕上的说明进行操作：
a. 您可以添加 --keywords 选项，以额外清理给定关键字列表。
b. 您可以添加 --usernames 选项以模糊处理进一步敏感的用户名。
```shell
[user@server1 ~]$ sudo sos clean /var/tmp/sos-collector-2022-05-15-pafsr.tar.xz
[sudo] password for user:

sos clean (version 4.2)

This command will attempt to obfuscate information that is generally considered to be potentially sensitive. Such information includes IP addresses, MAC addresses, domain names, and any user-provided keywords.

Note that this utility provides a best-effort approach to data obfuscation, but it does not guarantee that such obfuscation provides complete coverage of all such data in the archive, or that any obfuscation is provided to data that does not fit the description above.

Users should review any resulting data and/or archives generated or processed by this utility for remaining sensitive content before being passed to a third party.


Press ENTER to continue, or CTRL-C to quit.

Found 4 total reports to obfuscate, processing up to 4 concurrently

sosreport-primary-rhel8-2022-05-15-nchbdmd :      Extracting...
sosreport-sos-node1-2022-05-15-wmlomgu :      Extracting...
sosreport-sos-node2-2022-05-15-obsudzc :      Extracting...
sos-collector-2022-05-15-pafsr :                   Beginning obfuscation...
sosreport-sos-node1-2022-05-15-wmlomgu :      Beginning obfuscation...
sos-collector-2022-05-15-pafsr :                   Obfuscation completed
sosreport-primary-rhel8-2022-05-15-nchbdmd :      Beginning obfuscation...
sosreport-sos-node2-2022-05-15-obsudzc :      Beginning obfuscation...
sosreport-primary-rhel8-2022-05-15-nchbdmd :      Re-compressing...
sosreport-sos-node2-2022-05-15-obsudzc :      Re-compressing...
sosreport-sos-node1-2022-05-15-wmlomgu :      Re-compressing...
sosreport-primary-rhel8-2022-05-15-nchbdmd :      Obfuscation completed
sosreport-sos-node2-2022-05-15-obsudzc :      Obfuscation completed
sosreport-sos-node1-2022-05-15-wmlomgu :      Obfuscation completed

Successfully obfuscated 4 report(s)

A mapping of obfuscated elements is available at
    /var/tmp/sos-collector-2022-05-15-pafsr-private_map

The obfuscated archive is available at
    /var/tmp/sos-collector-2022-05-15-pafsr-obfuscated.tar.xz

    Size    157.10KiB
    Owner    root

Please send the obfuscated archive to your support representative and keep the mapping file private
```
验证步骤：
验证 sos clean 命令是否在与命令输出中描述匹配的 /var/tmp/ 目录中创建了模糊的存档和模糊的归档。
```shell
[user@server1 ~]$ sudo ls -l /var/tmp/sos-collector-2022-05-15-pafsr-private_map /var/tmp/sos-collector-2022-05-15-pafsr-obfuscated.tar.xz
[sudo] password for user:

-rw-------. 1 root root 160868 May 15 16:10 /var/tmp/sos-collector-2022-05-15-pafsr-obfuscated.tar.xz
-rw-------. 1 root root  96622 May 15 16:10 /var/tmp/sos-collector-2022-05-15-pafsr-private_map
```
检查 *-private_map 文件中的 obfuscation 映射：
```shell
[user@server1 ~]$ sudo cat /var/tmp/sos-collector-2022-05-15-pafsr-private_map
[sudo] password for user:

{
    "hostname_map": {
        "pmoravec-rhel8": "host0"
    },
    "ip_map": {
        "10.44.128.0/22": "100.0.0.0/22",
..
    "username_map": {
        "foobaruser": "obfuscateduser0",
        "jsmith": "obfuscateduser1",
        "johndoe": "obfuscateduser2"
    }
}
```
注意：保持原始的 unobfuscated 归档和 *private_map 文件，因为红帽支持可能会引用您需要转换为原始值的模糊术语。
# 4. 救援环境中生成 sos 报告
如果一个 Red Hat Enterprise Linux（RHEL）主机无法正确引导，您可以将主机引导至 救援环境 中来收集 sos 报告。
使用救援环境，您可以在 /mnt/sysimage 下挂载目标系统，访问其内容并运行 sos report 命令。
a. 从安装源引导主机。
b. 在安装介质的引导菜单中，选择 Troubleshooting 选项。
c. 在故障排除菜单中选择 Rescue a Red Hat Enterprise Linux system 选项。
d. 在 Rescue 菜单中，选择 1 并按 Enter 键继续，并在 /mnt/sysimage 目录下挂载系统。
e. 提示时按 Enter 键进行一个 shell。
f. 使用 chroot /mnt/sysimage 命令将救援会话的显式根目录改为 /mnt/sysimage。
g. 可选： 您的网络将不能在初始救援环境中启动，因此请确保首先建立它。例如，如果网络需要静态 IP 地址，并且您希望通过网络传输 sos 报告，请配置网络：
g1. 确定您要使用的以太网设备： 
```shell
$ ip link show
```
g2. 为网络接口分配一个 IP 地址，并设置默认网关。例如，如果您要将子网为 255.255.255.0 的 IP 地址 192.168.0.1（其 CIDR 为 24 ）添加到设备 enp1s0，请输入：
```shell
$ ip address add <192.168.0.1/24> dev <enp1s0>
$ ip route add default via <192.168.0.254>
```
g3. 向 /etc/resolv.conf 文件中添加一个 nameserver 条目，例如：
```shell
$ nameserver <192.168.0.5>
```
h. 运行 sos report 命令并按照屏幕的说明进行操作。
i. 记录下控制台输出末尾显示的 sos 报告文件名称。
j. 如果您的主机还没有连接到互联网，使用 scp 将 sos 报告传送到网络中的另一台主机。
k. 验证步骤
验证 sos 工具是否在 /var/tmp/ 目录中创建了一个存档。
