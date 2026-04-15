---
title: "Linux诊断和故障排除系列(十二) -- 日志持久化和转发"
description: 
date: 2024-07-12T20:02:29+08:00
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
  - 日志持久化和转发
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 日志文件
日志文件是包含有关系统的消息的文件，包括内核、服务及其上运行的应用。不同信息有不同的日志文件。
当尝试对系统问题进行故障排除（如尝试加载内核驱动程序或查找未授权登录尝试系统时）时，日志文件非常有用。
有些日志文件由名为 rsyslogd 的守护进程控制。rsyslogd 守护进程是 sysklogd 的增强替代品，提供扩展过滤、加密保护消息转发、各种配置选项、输入和输出模块，支持通过 TCP 或 UDP 协议传输。请注意，rsyslogd 与 sysklogd 兼容。
日志文件也可以由 journald 守护进程（ systemd 的一个组件）进行管理。journald 守护进程捕获 Syslog 消息、内核日志消息、初始 RAM 磁盘和早期启动消息，以及写入到所有服务的标准输出和标准错误输出的消息，对其进行索引，并使其可供用户使用。
## 1.1 查找日志文件
由 rsyslogd 维护的日志文件列表可在 /etc/rsyslog.conf 配置文件中找到。大多数日志文件都位于 /var/log/ 目录中。httpd 和 samba 等一些应用在其日志文件中有一个 /var/log/ 中的目录。

您可以在 /var/log/ 目录中看到多个文件，其编号位于 /var/log/ 目录中（例如： cron-20100906）。这些数据表示已添加到轮转的日志文件中的时间戳。日志文件会被轮转，使其文件大小不会变得太大。logrotate 软件包包含一个 cron 任务，它根据 /etc/logrotate.conf 配置文件和 /etc/logrotate.d/ 目录中的配置文件自动轮转日志文件。

# 2. journald
Journal 是 systemd 的一个组件，负责查看和管理日志文件。
日志记录数据由日志的 journald 服务收集、存储和处理。它基于从内核、用户进程、标准输出以及系统服务的标准错误输出或其原生 API 收到的日志信息来创建和维护名为日志的二进制文件。这些日志经过结构化和索引化，可提供相对较快的寻道时间。
journald: 是systemd的内存里的系统日志服务，也就是会记录从启动到关闭的日志。但如果重启，日志就都没有了。
使用journalctl命令访问日志。-f持续输出日志，-e跳到最后，-k只看kernel的日志，-u通过服务来查找，-p优先级
```shell
$ journalctl -k     //只看kernel的日志
$ journalctl -u sshd        //查看服务 sshd的相关日志
Apr 28 15:35:39 hecs-295729 systemd[1]: Starting OpenSSH server daemon...
Apr 28 15:35:39 hecs-295729 sshd[1167]: Server listening on 0.0.0.0 port 53975.
Apr 28 15:35:39 hecs-295729 sshd[1167]: Server listening on :: port 53975
$ journalctl -p 4       //显示优先级4及其以下的(0-4)的日志
$ journalctl -p emerg   //显示emerg优先级的日志
$ journalctl -xb        //显示自上次启动以来的所有日志
journalctl --since '2024-04-22 00:00:00' --until '2024-05-22 12:00:00'        //显示自从 什么时间， 直到什么时间 的日志。注意时间格式和上面的不一样
```
## .2.1 查看日志文件
若要访问日志，可使用 journalctl 工具，以 root 用户身份进行：
```shell
$ journalctl
```
减少 journalctl 输出的最简单方法是使用 -n 选项，它只列出指定数量的最新日志条目：
使用要显示的行数替换 Number。如果未指定数字，journalctl 将显示最新的十个条目。
```shell
$ journalctl -n Number
```
journalctl 命令允许使用以下语法控制输出格式：
使用指定所需输出格式的关键字替换 form。有多个选项，如 verbose （返回包含所有字段的全结构条目项）、export（ 创建 适合备份和网络传输的二进制流）和 json （将条目格式化为 JSON 数据结构）。有关关键字的完整列表，请查看 journalctl(1) 手册页。
```shell
$ journalctl -o verbose
```
## 2.2 过滤消息
按优先级过滤
要只查看优先级为 error 或更高的条目，请使用：
```shell
$ journalctl -p err
```
按时间过滤
要只从当前引导中查看日志条目，请键入：
```shell
$ journalctl -b
```
基于时间的过滤更有用：
使用 --since 和 --until 时，您只能查看在指定时间范围内创建的日志消息。
```shell
$ journalctl --since=value --until=value
```
您可以将值以日期和时间形式传递给这些选项
```shell
4 journalctl -p warning --since="2013-3-16 23:59:59"
```
## 2.3 高级过滤
有关 systemd 可存储的元数据的完整描述，请参阅 systemd.journal-fields(7) 手册页。为每个日志消息收集此元数据，无需用户干预。

要查看指定字段中出现的唯一值列表，请使用以下语法：
使用您感兴趣的字段的名称替换 fieldname。
```shell
$ journalctl -F fieldname
```
要只显示适合特定条件的日志条目，请使用以下语法：
使用字段名称替换 fieldname ，并将值替换为该字段中包含的特定值。因此，只会返回与这个条件匹配的行。
```shell
$ journalctl fieldname=value
```
要显示 user 下由 avahi-daemon.service 或 crond.service 创建的条目，请使用以下命令：
```shell
$ journalctl _UID=70 _SYSTEMD_UNIT=avahi-daemon.service _SYSTEMD_UNIT=crond.service
```

# 3. rsyslod
rsyslod: 是一个系统日志工具，用于过滤和管理日志，日志可以在/var/log中找到(增强了TCP的过滤和支持)
每一条信息都包括了2部分：Facilities和Priority
## 3.1 Facilities是提供日志信息的子系统
Facilities是提供日志信息的子系统。例如auth/authpriv/cron/daemon/kern/lpr/mail/mark/news/syslog/user/uucp/local0-local7
## 3.2 Priority是优先级
Priority是优先级，从0-7，严重程度递减。0 emerg/1 alert/2 crit/3 error/4 warn/5 notice/6 info/7 debug

#  4.  启用持久性存储
默认情况下，日志仅将日志文件 存储在 内存中或 /run/log/journal/ 目录中的小型环缓冲器中。此目录日志数据不会永久保存。使用默认配置时，syslog 会读取日志并将其存储在 /var/log/ 目录中。
启用持久日志记录后，日志文件存储在 /var/log/journal 中，这意味着它们会在重启后保留。然后日志可以替换某些用户的 rsyslog 
## 4.1 编辑/etc/systemd/journald.conf 
打开 /etc/systemd/journald.conf <<<<<< 使用man 5 journald.conf来查看配置文件里面的每一个选项的说明
把 Storage=auto 这一行注释给取消
## 4.2 手动创建日志目录
为日志文件创建一个位置 mkdir -p /var/log/journal/
设置权限 chown root:systemd-journal /var/log/journal
chmod 2755 /var/log/journal
## 4.3 重启 journald 以应用更改
重启服务 systemctl restart systemd-journald
验证，看看是否有日志输出 ll /var/log/journal/
## 4.4 journald.conf参数 
如果将 journald 配置为持久化保存日志，那么默认情况下，日志大小限制为该文件分区的 10%，最多可占用 4GB 磁盘空间，这是通过修改参数 SystemMaxUse 来调整的
```shell
# /etc/systemd/journald.conf
RuntimeMaxUse= 日志在内存里最大容量
SystemMaxUse= 日志最大可用容量，然后对日志文件执行systemd-journald logrotate。默认为分配给节点的总物理内存的10％
RuntimeMaxFileSize= 在内存中单个日志文件的最大容量
SystemMaxFileSize= 单个日志文件的最大容量 到达此限制后日志文件将会自动滚动。 默认值是对应的 SystemMaxUse=/RuntimeMaxUse= 值的1/8 ， 这也意味着日志滚动 默认保留7个历史文件。
RuntimeKeepFree/SystemKeepFree= 控制systemd-journald将为其他用途保留多少磁盘空间，之后将对日志文件执行systemd-journald logrotate。默认为分配给节点的总物理内存的15％
SystemMaxFiles/RuntimeMaxFiles= 限制最多允许同时存在多少个日志文件， 超出此限制后， 最老的日志文件将被删除， 而当前的活动日志文件 则不受影响。 默认值为100个。
MaxRetentionSec=日志滚动的时间间隔。通常并不需要使用基于时间的日志滚动策略， 因为由SystemMaxFileSize/RuntimeMaxFileSize= 控制的基于文件大小的日志滚动策略已经可以确保日志文件的大小不会超标。 默认值是一个月， 设为零表示禁用基于时间的日志滚动策略。
MaxRetentionSec=日志文件的最大保留期限。 当日志文件的最后修改时间(mtime)与当前时间之差，大于此处设置的值时，日志文件将会被删除。 通常并不需要使用基于时间的日志删除策略。
```
# 5. 远程日志转发
rsyslog 服务提供运行日志记录服务器和将各个系统配置为将其日志文件发送到日志记录服务器的功能。
## 5.1 安装rsyslog(在日志服务器和日志客户端上)
```shell
$ yum install rsyslog		//安装rsyslog
```
系统日志流量的默认协议和端口为 UDP 和 514，如 /etc/services 文件中列出的。但是，r syslog 默认为在端口 514 上使用 TCP。在 配置文件 /etc/rsyslog.conf 中，TCP 由 @@ 表示。
验证 rsyslog 正在侦听哪些端口：
```shell
$ netstat -tnlp | grep rsyslog
tcp    0   0 0.0.0.0:10514      0.0.0.0:*  LISTEN   2528/rsyslogd
tcp    0   0 :::10514        :::*    LISTEN   2528/rsyslogd
```
## 5.2 配置日志服务器(在日志服务器上)
### 5.2.1 方法一 (redhat官方文档)将 rsyslog 配置为接收和排序远程日志消息
https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/7/html/system_administrators_guide/s1-configuring_rsyslog_on_a_logging_server
使用rsyslog来配置远程日志
在 modules 部分下方，在Provides UDP syslog 上方，添加这些行
```shell
# Define templates before the rules that use them

# Per-Host Templates for Remote Systems #
$template TmplAuthpriv, "/var/log/remote/auth/%HOSTNAME%/%PROGRAMNAME:::secpath-replace%.log"
$template TmplMsg, "/var/log/remote/msg/%HOSTNAME%/%PROGRAMNAME:::secpath-replace%.log"
```
将默认的 Provides TCP syslog 接收 部分替换为以下内容：
```shell
# Provides TCP syslog reception
$ModLoad imtcp
# Adding this ruleset to process remote messages
$RuleSet remote1
authpriv.*  ?TmplAuthpriv
*.info;mail.none;authpriv.none;cron.none  ?TmplMsg
$RuleSet RSYSLOG_DefaultRuleset  #End the rule set by switching back to the default rule set
$InputTCPServerBindRuleset remote1 #Define a new input and bind it to the "remote1" rule set
$InputTCPServerRun 10514
```
重启rsyslog
```shell
$ systemctl restart rsyslog
$ systemctl status rsyslog
```
### 5.2.2 方法二(RH342课程里的方法)
在/etc/rsyslog.conf，取消下面最后二行的注释(在 # Provides TCP syslog reception 下面)
```shell
$ vi /etc/rsyslog.conf
# Provides TCP syslog reception
$ModLoad imtcp
$InputTCPServerRun 514
```
编辑要记录的日志，通过Facilities和Priorities来编辑：
```shell
mail.*	/var/log/maillog		<<<先是acilities子系统，然后是Priorities优先级(*代表所有)，最后是日志放哪里
*.emerg		:omusrmsg:*				<<<把 所有子系统的 emerg类信息，都直接发给当前所有登录用户
auth;syslog.crit;error  .				<<<<多个子系统入一行。auth的所有都记录，syslog的只记录crit的，error的都记录。并把日志 记录在根目录下

# log all
$template LogLocale,"/var/log/hosts/%HOSTNAME%.log"			<<<定义一个模板，名字叫LogLocale。这样，每个主机都会以自己的主机名来命名了
*.*		-?LogLocale						<<<引用名字为LogLocale的模板，引用方法为 -?
```
重启rsyslog
```shell
$ systemctl restart rsyslog
$ systemctl status rsyslog
```
## 5.3 配置日志客户端(在客户端_要收集日志的机器上)
### 5.3.1 方法一 (redhat官方文档)例 23.12. 可靠将日志消息转发至服务器
https://docs.redhat.com/zh_hans/documentation/red_hat_enterprise_linux/7/html/system_administrators_guide/s1-working_with_queues_in_rsyslog#ex-net_forwarding_with_queue
假设任务是将日志消息从系统转发到主机名为 example.com 的服务器，并且配置操作队列以缓冲消息（以防服务器中断）。
在 /etc/rsyslog.conf 中使用 以下配置，或在 /etc/rsyslog.d/ 目录中创建包含以下内容的文件：
```shell
. action(type=”omfwd”
queue.type=”LinkedList”
queue.filename=”example_fwd”
action.resumeRetryCount="-1"
queue.saveonshutdown="on"
Target="example.com" Port="6514" Protocol="tcp")
```
其中：
queue.type 启用 LinkedList 内存中队列，
queue.filename 定义磁盘存储，在这种情况下，备份文件在 /var/lib/rsyslog/ 目录中创建，前缀为 example_fwd。
action.resumeRetryCount= "-1" 设置阻止 rsyslog 在重试连接时丢弃消息（如果服务器没有响应）
如果 rsyslog 关闭，启用的 queue.saveonshutdown 可保存内存中数据。
最后一行使用可靠的 TCP 发送将所有收到的消息转发到日志记录服务器，端口规格是可选的。

使用上述配置时，如果远程服务器无法访问，rsyslog 将消息保留在内存中。只有在 rsyslog 用尽配置的内存队列空间或需要关闭时，才会在磁盘上创建文件，这将提高系统性能。
### 5.3.2 方法二(RH342课程里的方法)
使用rsyslog来配置远程日志
在/etc/rsyslog.conf，在文件最下面，编辑最下面一段(在 # ### begin forwarding rule ### 下面)
```shell
$ vi /etc/rsyslog.conf
# ### begin forwarding rule ###
# The statement between the begin ... end define a SINGLE forwarding
# rule. They belong together, do NOT split them. If you create multiple
# forwarding rules, duplicate the whole block!
# Remote Logging (we use TCP for reliable delivery)
#
# An on-disk queue is created for this action. If the remote host is
# down, messages are spooled to disk and sent when it is up again.
$ActionQueueFileName fwdRule1 # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on # save messages to disk on shutdown
$ActionQueueType LinkedList   # run asynchronously
$ActionResumeRetryCount -1    # infinite retries if host is down
# remote host is: name/ip:port, e.g. 192.168.0.1:514, port optional
*.* @@remote-host:514           //在这里把remote-host改为上面日志服务器的IP，把514改为上面日志服务器中配置文件中的端口号(上一步里面)
# ### end of the forwarding rule ###
```
重启rsyslog
```shell
$ systemctl restart rsyslog
$ systemctl status rsyslog
```
测试
```shell
$ logger hihi hello         //发送一条日志。然后在日志服务器上应该会看到
```









