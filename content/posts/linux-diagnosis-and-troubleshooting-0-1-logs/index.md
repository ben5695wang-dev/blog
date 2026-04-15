---
title: "Linux诊断和故障排除(0-1) — 日志"
description: 
date: 2026-06-30T20:02:29+08:00
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
  - 日志
build:
    list: always    # Change to "never" to hide the page from the list
---

# 日志
## rsyslod
rsyslod是一个系统日志工具，用于过滤和管理日志，日志可以在/var/log中找到(增强了TCP的过滤和支持)
	每一条信息都包括了2部分：Facilities和Priority
### Facilities	
Facilities是提供日志信息的子系统。例如auth/authpriv/cron/daemon/kern/lpr/mail/mark/news/syslog/user/uucp/local0-local7
### Priority	
Priority是优先级，从0-7，严重程度递减。0 emerg/1 alert/2 crit/3 error/4 warn/5 notice/6 info/7 debug

# journald
journald: 是systemd的内存里的系统日志服务，也就是会记录从启动到关闭的日志。<font color=red>但如果重启，日志就都没有了。</font>
使用journalctl命令访问日志。-f持续输出日志，-e跳到最后，-k只看kernel的日志，-u通过服务来查找，-p优先级

**<font color=red>journalctl -D 可以查看指定dir里面的日志
journalctl --file 可以查看指定文件的日志</font>**
```shell
$ systemctl list-units --type=service --all				//列出所有服务（无论它们是已加载还是不活动）
$ systemctl list-units --type=service							//列出每个已加载的正在运行、活动或失败的服务
$ systemctl list-units --type=service --state=running		//统中已加载和正在运行的服务
$ systemctl list-unit-files --state=enabled			//系统中启用的服务

$ journalctl -u systemd-logind.service              // -u可以接收服务名作为参数
$ journalctl -u systemd-logind.service -p 4         // -p可以接收优先级作为参数，从0-7，严重程度递减。0 emerg/1 alert/2 crit/3 error/4 warn/5 notice/6 info/7 debug
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

# 配置持久journald日志
1. 打开 /etc/systemd/journald.conf
2. 把 Storage=auto 这一行注释给取消
3. 为日志文件创建一个位置 mkdir /var/log/journal
4. 设置权限 chown root:systemd-journal /var/log/journal
		chmod 2755  /var/log/journal
5. 重启服务 systemctl restart systemd-journald
6. 验证，看看是否有日志输出 ll /var/log/journal/

# 配置日志转发到远程服务器
## 在日志服务器上
使用rsyslog来配置远程日志
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
## 在客户端_要收集日志的机器上
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