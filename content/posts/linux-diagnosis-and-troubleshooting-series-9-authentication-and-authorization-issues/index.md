---
title: "Linux诊断和故障排除系列(九) -- 身份验证和授权问题诊断"
description: 
date: 2024-07-09T20:02:29+08:00
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
  - 身份验证和授权问题诊断
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 识别并修复PAM问题
PAM(Pluggable Authentication Module)
## 1.1 PAM基本配置
```shell
$ cat /etc/sysconfig/authconfig			//遗留的PAM系统配置接口
PASSWDALGORITHM=sha512
USESHADOW=yes
$ cat /etc/nsswitch.conf						//Name service配置
passwd:     files sss
shadow:     files sss
group:      files sss
$ cat /etc/authselect/*							//RHEL8中有的，取代authconfig的
```
## 1.2 PAM模块
```shell
$ cat /etc/pam.d/*				//PAM的模块配置，对应各服务名
$ cat /etc/pam.d/login
account    required     pam_nologin.so		//从左向右依次是类型、控制、模块、参数
```
类型，对PAM的调用类型，可以是auth、account、password、session
控制，PAM如何处理所提供的类型的请求，required代表任何授权类型的请求都要有一个必要的控制，并对这要求的控制作出回应
模块，PAM使用的模块，也就是响应上一个控制的方式。
参数，与PAM模块有关的额外的参数
### 1.2.1 类型：
对PAM的调用类型
    auth：验证一个用户的凭据，即这是正确的用户吗？
    account：确认账户的存在和访问，即可以访问系统中的什么
    password：管理和更新密码，和改变与认证的连接
    session：对用户的会话管理，像SELinux、审计跟踪等
### 1.2.2 控制
PAM如何处理请求，是否通过整个模块。
    required：通过所有模块，如果失败了，就是失败了。
    requisite：如果失败，就会立即停止，该模块不会继续
    sufficient：只要满足它的基本要求，就可以运行这个模块
    optional：任何模块或信息并不是实际用于认证的，只是一些额外的数据可能要被提供
    include：配置中的所有行都必须被包含，如果失败，则立即失败
    substack：配置中的所有行都必须被包含，当模块遇到done或die时，只有substack结束，而不是整个模块
```shell
$ cat /etc/pam.d/system-auth
$ ls /usr/share/doc/pam/txts			//这个目录里每个文件对应一个模块的说明 
$ vi /usr/share/doc/pam-1.1.8/txts/README.pam_rhost		//pam_rhost模块的说明，包括描述、选项、例子等
$ ls /lib64/security/pam_*				//查看哪个模块在当前服务器上是可用的
$ dnf install samba samba-client			//安装smb
$ systemctl start smb						
$ smbpasswd -a cloud_user							//给cloud_user这个用户一个smb密码
$ cat /etc/pam.d/samba								//查看规则
$ cat /usr/share/doc/pam/txts/REAME.pam_nologin		//查看对应规则说明 
$ vi /etc/samba/smb.conf							//查看smb配置文件
[global]
	obey pam restrictions = yes				//添加这一行是为了让smb服从pam限制
$ systemctl restart smb
$ smbclient -U cloud_user -L localhost		//使用cloud_user登录本机的smb，提供密码后成功
$ tail /var/log/secure				//查看登录信息，这里的日志对于PD没什么太大帮助
$ vi /etc/pam.d/samba					//破坏
删除一些内容
$ smbclient -U cloud_user -L localhost		//再次登录，失败: session setup failed: NT_STATUS_LOGIN_FAILURE
$ rpm -V samba								//先检查samba包的完整性
S.5....T. c /etc/pam.d/samba		//看着不太对
$ cat /etc/pam.d/samba					//果然不对
$ mv /etc/pam.d/samba /etc/pam.d/samba.old
$ dnf reinstall samba
$ smbclient -U cloud_user -L localhost		//再次登录，OK
```
# 2. 识别并修复LDAP和Kerberos身份管理问题
## 2.1 RHEL7和8的改变
在RHEL8，authconfig is dead，OpenLDAP也不再支持
authselect是替换authconfig的。authselect提供了比authconfig更有限的访问，还鼓励使用配置文件来配置PAM而不是自己配置。也就是说还要手动配置那些配置文件。
## 2.2 sssd
SSSD（System Security Services Daemon）是一个守护进程，用于Linux系统，提供集中式身份管理和认证服务。它允许本地系统连接到外部身份验证和目录服务，例如LDAP、Active Directory、Kerberos等，以实现用户和组信息的同步和认证。
当我们设置OpenLDAP和Kerberos配置文件时，要用到sssd和sssd.conf文件，因为sssd(daemon)是到OpenLDAP和Kerberos的连接
/etc/sssd/sssd.conf //System Security Services Daemon configuration file
为了选择适当的配置文件为我们的认证设置使用，需要使用authselect命令。
## 2.3 authselect
authselect //配置、更新系统认证配置文件的工具
currect //显示当前使用的配置文件
select //更新(设置)配置文件
apply-changes //更新当前正在使用的配置文件
```shell
$ authselect current				//查看当前配置文件
$ authselect select sssd		//修改为使用sssd
$ vi /etc/sssd/ssd.conf			//查看当前配置文件的配置
$ authselect apply-changes				//保存修改
```

