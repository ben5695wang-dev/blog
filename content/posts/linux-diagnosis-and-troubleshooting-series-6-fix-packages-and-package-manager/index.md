---
title: "Linux诊断和故障排除系列(六) -- 修复软件包及管理器"
description: 
date: 2024-07-06T20:02:29+08:00
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
  - 修复软件包及管理器
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 解决包管理相关性问题
## 1.1 包管理器是如何工作的？
a. 在包管理器以前，都是自己下载tar包。
b. RPM保存了一个软件包的数据库并跟踪这些软件包的变化，而RPM是不检查包的依赖性的。
c. YUM/DNF负责检查软件包的依赖性，他们使用自己的YUM/DNF数据库来检查依赖性，并将依赖性和软件包关联起来。YUM/DNF也会使用RPM数据库。
d. DNF就是下一代的YUM，他解决了YUM的一些问题，并解决了YUM占用过多内存的问题。
## 1.2 yum/dnf命令
yum list --show-duplicates httpd //查看软件包有多少个版本
yum deplist httpd //检查依赖性
yum provides /etc/httpd/conf/httpd.conf  //显示哪个安装的包提供了这个文件
yum downgrade httpd-2.2.3-22.el5 //降级(回退)到之前版本
```shell
$ yum list --show-duplicates httpd          //检查httpd有多少个版本
Loaded plugins: fastestmirror, product-id, search-disabled-repos, subscription-manager
Loading mirror speeds from cached hostfile
Available Packages
httpd.x86_64                                                                                                  2.4.6-95.el7.centos                                                                                                     base   
httpd.x86_64                                                                                                  2.4.6-97.el7.centos                                                                                                     updates
httpd.x86_64                                                                                                  2.4.6-97.el7.centos.1                                                                                                   updates
httpd.x86_64                                                                                                  2.4.6-97.el7.centos.2                                                                                                   updates
httpd.x86_64                                                                                                  2.4.6-97.el7.centos.4                                                                                                   updates
httpd.x86_64                                                                                                  2.4.6-97.el7.centos.5                                                                                                   updates
httpd.x86_64                                                                                                  2.4.6-98.el7.centos.6                                                                                                   updates
httpd.x86_64                                                                                                  2.4.6-98.el7.centos.7                                                                                                   updates
httpd.x86_64                                                                                                  2.4.6-99.el7.centos.1                                                                                                   updates
$ yum install httpd             //安装httpd
$ yum deplist httpd               //显示httpd依赖哪些包
$ yum provides /etc/httpd/conf/httpd.conf               //显示哪个软件包或哪些软件包提供了这个文件
Loaded plugins: fastestmirror, product-id, search-disabled-repos, subscription-manager
Loading mirror speeds from cached hostfile
httpd-2.4.6-95.el7.centos.x86_64 : Apache HTTP Server
Repo        : base
Matched from:
Filename    : /etc/httpd/conf/httpd.conf

httpd-2.4.6-97.el7.centos.x86_64 : Apache HTTP Server
Repo        : updates
Matched from:
Filename    : /etc/httpd/conf/httpd.conf

httpd-2.4.6-97.el7.centos.1.x86_64 : Apache HTTP Server
Repo        : updates
Matched from:
Filename    : /etc/httpd/conf/httpd.conf
......
```
## 1.3 包依赖性相关命令
如果因为依赖性问题导致软件包不能安装或升级，使用以下命令：
dnf deplist //检查依赖性
dnf provides //显示哪个安装的包提供了这个文件
dnf versionlock list //查看是否有锁定的软件包
## 1.4 软件版本锁定
版本锁定并不是YUM/DNF自带的，要安装第三方软件包，从而让我们把软件包锁定在一个指定版本
dnf-plugin-versionlock
dnf-plugin-versionlock list //显示锁定的软件包
dnf-plugin-versionlock add //添加一个软件包，从而锁定
dnf-plugin-versionlock delete //删除一个软件包，从而不再锁定
dnf-plugin-versionlock clear //删除所有软件包，从而不再锁定任何软件包
```shell
$ dnf install dnf-plugin-versionlock		//安装
$ dnf versionlock list		//显示锁定的软件包
$ dnf versionlock add httpd		//添加httpd软件包，从而锁定，并在当前版本号上
$ dnf versionlock list		//显示锁定的软件包
$ dnf versionlock delete httpd		//删除锁定的httpd软件包
```

# 2. 恢复损坏的RPM数据库 
当运行RPM等命令失败时，可能是RPM DB出问题了，解决步骤如下：
第一步：使用lsof来检查所有与RPM相关的被打开中的文件，并清除与RPM相关的锁
```shell
lsof | grep /var/lib/rpm
rm -rf /var/lib/rpm/__db*
```
第二步：使用/usr/lib/rpm/rpmdb验证RPM DB是否损坏
```shell
/usr/lib/rpm/rpmdb_verify /var/lib/rpm/Packages
```
第三步：备份RPM DB或整个RPM目录
```shell
mv /var/lib/rpm/Packages /var/lib/rpm/Packages.bak
```
第四步：Dump and Load
使用/usr/lib/rpm/rpmdb_dump来转储备份文件，然后使用/usr/lib/rpm/rpmdb_load来加载一个新的
```shell
/usr/lib/rpm/rpmdb_dump /var/lib/rpm/Packages.bak | /usr/lib/rpm/rpmdb_load /var/lib/rpm/Packages
```
第五步：再次验证
使用/usr/lib/rpm/rpmdb验证RPM DB是否损坏，这次验证的是新的包文件
```shell
/usr/lib/rpm/rpmdb_verify /var/lib/rpm/Packages
```
第六步：重建DB
使用rpm -v --rebuilddb 重建DB
```shell
rpm -vv --rebuilddb
```
## 2.1 例子
```shell
$ cd /var/lib/rpm					//这个目录下的Packages文件就是rpm db所在
$ dnf install hexedit -y
$ hexedit Packages 				//随机修改一些数字，来破坏Packages
$ lsof | grep /var/lib/rpm		//查看是否有和/var/lib/rpm/Packages相关的被打开的文件，如果有，就用kill -9杀其进程
$ kill -9 4553						//用kill -9杀其进程
$ lsof | grep /var/lib/rpm		//再次确认已经没有和/var/lib/rpm/Packages相关的被打开的文件
$ ls /var/lib/rpm					//查看这个目录下是否有 __db.001 这样的文件，也就是锁
$ rm -rf /var/lib/rpm/__db*						//删除__db.001 这样的文件，也就是删除锁
$ ls /var/lib/rpm					//再次确认这个目录下没有 __db.001 这样的文件，也就是锁
$ ls /usr/lib/rpm					//这里面都是与RPM相关的可以使用的工具
macros  macros.d  platform  rpm2cpio.sh  rpm.daily  rpmdb_dump  rpmdb_load  rpmdb_loadcvt  rpmdb_recover  rpmdb_stat  rpmdb_upgrade  rpmdb_verify  rpm.log  rpmpopt-4.11.3  rpmrc  rpm.supp  tgpg			//最重要的就是rpmdb_verify，rpmdb_dump，rpmdb_load 
$ /usr/lib/rpm/rpmdb_verify /var/lib/rpm/Packages       //全路径，验证PRM DB，如果损坏会报Verfifcation of Packages failed
$ mv /var/lib/rpm/Packages /var/lib/rpm/Packages.bak		//备份RPM DB
$ /usr/lib/rpm/rpmdb_dump /var/lib/rpm/Packages.bak | /usr/lib/rpm/rpmdb_load /var/lib/rpm/Packages       //使用备份的Packages.bak，用rpmdb_dump和rpmdb_load创建一个新的RPM DB文件，先用rpmdb_dump从备份的RPM DB文件中转储，然后用rpmdb_load恢复一个新的RPM DB 文件
$ /usr/lib/rpm/rpmdb_verify /var/lib/rpm/Packages       //再次验证PRM DB，一切正常
$ rpm -vv --rebuilddb				//重建RPM DB
```
# 3. 追踪rpm文件变化并恢复更改的rpm文件
## 3.1. 追踪rpm文件变化
rpm -Va //验证所有rpm(安装的)相关文件
输出的第一列中的字母代表：
S: Size,这个文件的大小被修改了
M: Mode,这个文件的权限或类型被修改了
5: MD5,这个文件的内容被修改了
rpm不会跟踪配置文件的其内容变化！！！因为后面多个小c，代表这是一个配置文件
D: 设备的主/次号不匹配
L: Symlinks,这个文件的Symlinks被修改了
U: User,这个文件的所有者被修改了
G: Group,这个文件的所属组被修改了
T: Modification time,这个文件的修改时间被修改了
rpm -V //验证指定的包及其相关文件
rpm -qf //确定一个文件是哪个rpm包提供的
## 3.1.1 例子
```shell
$ dnf install httpd
$ dnf reinstall htppd		//重新安装一次，以便 确认任何改动都被重置了
$ cd /etc/httpd/conf
$ ll				//查看权限和属组
$ chmod 000 httpd.conf			//破坏
$ chown cloud_user:cloud_user httpd.conf		//破坏
$ vi /usr/lib/systemd/system/httpd.service		//对应的服务的文件
Hello												//随机添加内容，破坏
//现在启动httpd将会失败！
$ rpm -Va
S.5....T.  c /etc/security/limits.conf
......
$ rpm -V httpd
.M...UG..		c /etc/httpd/conf/httpd.conf                    //这个文件的权限、所有者和属组都被修改了。！！！这是配置文件，所以rpm不会跟踪其内容变化！！！后面多个小c，代表这是一个配置文件
S.5....T.			/usr/lib/systemd/system/httpd.service       //这个文件的大小、内容和修改时间都变了
```
## 3.2 恢复更改的rpm文件
rpm --setperms //恢复权限
rpm --setguids //恢复所有者和属组
如果以上这两个命令不行，就只能用yum/dnf重新安装了
dnf reinstall //重新安装，但这样做不会改变现在的配置文件的内容，只会确保其修复的内容
```shell
$ rpm -V httpd
.M...UG..		c /etc/httpd/conf/httpd.conf                    //这个文件的权限、所有者和属组都被修改了。！！！这是配置文件，所以rpm不会跟踪其内容变化！！！后面多个小c，代表这是一个配置文件
S.5....T.			/usr/lib/systemd/system/httpd.service       //这个文件的大小、内容和修改时间都变了
$ rpm --setperms httpd 		                                    //恢复权限
$ rpm -V httpd
.....UG..		c /etc/httpd/conf/httpd.conf                    //没有M了其内容变化！！！后面多个小c，代表这是一个配置文件
S.5....T.			/usr/lib/systemd/system/httpd.service       //这个文件的大小、内容和修改时间都变了
$ rpm --setguids httpd	                                        //恢复所有者和属组
$ rpm -V httpd
S.5....T.			/usr/lib/systemd/system/httpd.service       //这个文件的大小、内容和修改时间都变了
........P			/usr/sbin/suexec                            //这条可以忽略
$ dnf reinstall httpd-y                                         //只好重新安装
$ rpm -V httpd                                                  //输出为空，代表正常了！
```
