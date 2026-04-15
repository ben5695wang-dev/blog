---
title: "Linux诊断和故障排除系列(四) -- 修复文件系统"
description: 
date: 2024-07-04T20:02:29+08:00
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
  - 修复文件系统
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 修复损坏的文件系统
## 1.0 损坏文件系统
```shell
lsblk //查看磁盘
fdisk /dev/nvme1n1 //对nvme1n1划分 分区
mkfs.xfs /dev/nvme1n1p1 //对nvme1n1p1分区创建xfs文件系统
mkfs.ext4 /dev/nvme1n1p2 //对nvme1n1p2分区创建ext4文件系统
mkdir /store1 /store2 //创建挂载目录
mount -t xfs /dev/nvme1n1p1 /store1 //挂载FS
mount -t ext4 /dev/nvme1n1p2 /store2
echo 'hello' > /store1/hello
echo 'hello' > /store1/hello
umount /store*
// 将2个FS上面的数据破坏掉，并将FS类型改为swap
dd if/dev/urandom of=/dev/nvme1n1p1 bs=4096 count=10 seek=10000 && mkswap /dev/nvme1n1p1
dd if/dev/urandom of=/dev/nvme1n1p2 bs=4096 count=10 seek=10000 && mkswap /dev/nvme1n1p2
```
## 1.1 xfs
### 1.1.1 命令
xfs_repair 修复XFS的FS
-n 只检查，不修复
-L 删除jouranl log(日志)(如果日志被破坏了)
### 1.1.2 修复
所以，先运行xfs_repaire，如果不行，就用带-L参数的xfs_repair
xfs_repair -n /dev/nvme1n1p1 //这时会报 主超级块已经损坏，但可以找到第二超级块，但在-n模式下不能修改主超级块
xfs_repair /dev/nvme1n1p1 //修复
mount -t xfs /dev/nvme1n1p1 /store1 //一切正常了
## 1.2 ext4
### 1.2.1 命令
ext系列FS提供了e2fsck来修复ext-based的FS
-n 以只读方式打开FS，但不进行修复
-p 自动修复
dumpe2fs 显示定义的ext4 FS信息，包括超级块的信息
e2fsck /dev/nvme1n1p2 //这时会报 主超级块、inode等已经损坏，问要不要修复，一路yes去修复
mount -t ext4 /dev/nvme1n1p2 /store2 //一切正常了
```shell
$ dumpe2fs /dev/vda1 | more               //显示这个LV的ext4 FS的超级块等信息
dumpe2fs 1.42.9 (28-Dec-2013)
Filesystem volume name:   <none>
Last mounted on:          /
Filesystem UUID:          7c5d72cf-4d6b-4cd2-ac0d-fcc9270be4c4
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
......
$ dumpe2fs /dev/vda1 | grep superblock      //显示超级块信息
dumpe2fs 1.42.9 (28-Dec-2013)
  Primary superblock at 0, Group descriptors at 1-5                 //主超级块在inode 0这里
  Backup superblock at 32768, Group descriptors at 32769-32773
  Backup superblock at 98304, Group descriptors at 98305-98309
  Backup superblock at 163840, Group descriptors at 163841-163845
  Backup superblock at 229376, Group descriptors at 229377-229381
  Backup superblock at 294912, Group descriptors at 294913-294917
  Backup superblock at 819200, Group descriptors at 819201-819205
  Backup superblock at 884736, Group descriptors at 884737-884741
  Backup superblock at 1605632, Group descriptors at 1605633-1605637
  Backup superblock at 2654208, Group descriptors at 2654209-2654213
  Backup superblock at 4096000, Group descriptors at 4096001-4096005
  Backup superblock at 7962624, Group descriptors at 7962625-7962629
$ dd if/dev/zero of=/dev/nvme1n1p2 seek=32768 bs=1 count=2          //毁坏第2个超级块
$ e2fsck -p /dev/nvme1n1p2 -b 163840                                //指定使用163840这里的超级块来修复这个FS
```
# 2. LVM配置
相关基础命令
```shell
pvcreate /dev/nvme2n1 //创建一个PV
vgcreate vgstorage /dev/nvme2n1 //在/dev/nvme2n1这个PV上创建一个名为vgstorage的VG
lvcreate -L 1G -n lvstore vgstorage //在vgstorage这个VG上创建一个大小为1G名为lvstore的LV
mkfs.xfs /dev/vgstorage/lvstore //在lvstore这个LV上创建一个FS
mkdir /storage
blkid //查看逻辑卷的uuid
/dev/mapper/vgstorage-lvstore: UUID="7c5d72cf-4d6b-4cd2-ac0d-fcc9270be4c4" BLOCK_SIZE="512" TYPE="xfs"
vi /etc/fstab //在文件底部追加
UUID=7c5d72cf-4d6b-4cd2-ac0d-fcc9270be4c4 /storage xfs defaults 0 0
mount /storage
echo "hello" > /storage/hello
```
## 2.1 LVM archive
打开LVM archive: 在/etc/lvm/lvm.conf中修改archive = 1 来打开LVM archive
LVM archive位置: /etc/lvm/archive
查看可用archive: vgcfgrestore -l vgstorage //显示 vgstorage 这个VG的可用archive文件
umont /storage
### 2.1.1 创建LVM损坏
```shell
lvresize -L 100M /dev/vgstorage/lvstore //缩写LV
lvchange -an /dev/vgstorage/lvstore // disable lv
lvchange -ay /dev/vgstorage/lvstore // enable lv
mount /storage //报超级块错
```
## 2.2 vgcfgrestore恢复VG的元数据
vgcfgrestore 恢复virtual group的元数据
-l 显示可用archives
-f 从备份文件恢复
archive文件都是在对LV或VG执行命令之后生成的，所以找到有问题的命令的那个archive文件就可以了
vgcfgrestore -f /etc/lvm/archive/vgstorage_0002-1793243533.vg vgstorage //回答yes
lvchange -an /dev/vgstorage/lvstore // disable lv
lvchange -ay /dev/vgstorage/lvstore // enable lv
mount /storage //一切正常了
# 3. iSCSI问题
## 3.1 配置和检查iSCSI
有2台服务器，一台Target服务器，一台Initiator服务器。
在Target服务器上安装targetcli，在Initiator服务器上安装iscsiadm，并用iscsiadm与target进行交互。可以有多个Initiator，但Initiator与Target是一对一工作的。
![](Linux_pd_04_iSCSI_review.png)
在Target服务器上
```shell
$ dnf install targetcli						//安装targetcli
$ systemctl enable --now target		//启动targetcli
$ targetcli												//进入targetcli，才能与iSCSI互动
//在Target服务器上的targetcli中
/> ls                                       //查看iSCSI，现在没有
/> backstores/block create lun0 /dev/nvme1n1			//在/dev/nvme1n1上创建backstore lun0
/> iscsi/ create																	//创建iSCSI目标
/> ls iscsi/																			//现在有iqn了
/> iscsi/iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799/tpg1/             //进入这个位置
/> luns/ create /backstores/block/lun0     //在iqn内，将blockstorage与这个位置关联起来
/> exit                                    //退出targetcli，这会同时保存修改
```
在Initiator服务器上
```shell
$ dnf install iscsiadm		//安装iscsiadm
$ cat /etc/iscsi/initiatorname.iscsi		//查看InitatorName，并复制
InitiatorName=iqn.1994-05.com.redhat:27659b89e3ca
```
回到Target服务器上的targetcli中
```shell
/> iscsi/iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799/tpg1/             //进入这个位置
/> acls/create iqn.1994-05.com.redhat:27659b89e3ca                                  //创建一个ACL
/> exit                                    //退出targetcli，这会同时保存修改
$ ss -lntp		//找端口号是3260的LISTEN，这是默认的iSCSI进程
```
回到在Initiator服务器上
```shell
$ telnet Target_IP 3260         //通过Telnet来测试端口
Trying 172.31.101.193...
Connected to 172.31.101.193.
Escape character is '^]'.       //可以成功连接，强制退出吧

$ iscsiadm -m discovery -t sendtargets -p 172.31.101.193        //执行一个discovery，从而让iSCSI Initiator发现iSCSI Target的位置
172.31.101.193:3260,1 iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799
$ iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799 -l       //使用这个IQN，从initiator登录，这个两个服务器就连接上了。
                                                                //这里没有要用户名和密码之类的，是因为没有设置
$ vi /etc/iscsi/iscsid.conf			//检查认证设置，在Initiator使用iscsiadm -m discovery时，会使用这个文件里的用户名和密码
node.session.auth.authmethod = CHAP		//是否开启了CHAP
node.session.auth.username = username		//设置用户名
node.session.auth.password = password		//设置密码
```
回到Target服务器上的targetcli中
```shell
/> iscsi/iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799/tpg1/             //进入这个位置
/> get attribute                                                 //获取属性。里面的authenticatoin=0，也就是没有设置认证。也就是只要添加了Initiator的ACL，Initiator就可以访问Target
/> get att                                                       //获取认证相关信息，现在里面的userid和password都是空
/> set attribute authentication=1		//开启认证
/> set auth userid=initiator					//设置认证的用户名
/> set auth password=thisisagoogdpass					//设置认证的密码
/> exit
```
回到在Initiator服务器上
```shell
$ vi /etc/iscsi/iscsid.conf			//编辑认证设置
node.session.auth.authmethod = CHAP		//开启了CHAP
node.session.auth.username = initiator		//设置用户名
node.session.auth.password = thisisagoogdpass		//设置密码   
```
## 3.2 诊断iSCSI问题
### 3.2.1 诊断iSCSI问题步骤
先检查Target
systemctl status target //检查服务状态
iscsi/iqn.../tpg1/portal //检查配置，端口号等。这一步在Target和initiator上都执行
iscsi/iqn.../tpg1/acls //检查配置，ACL等
get attribute authenticatoin //检查认证是否打开
get auth //检查认证里面的参数

再检查Initiator
telnet 3260 //检查是否可以通过3260连接Target
iscsiadm -m session //检查是否有活动的session
iscsiadm -m discovery -t sendtargets -p :3260 //发现Target
iscsiadm -m node -o delete //删除discovery，然后重新创建
/etc/iscsi/iscsid.conf //检查认证设置
##3 3.2.2 例子
在Initiator服务器上
```shell
$ iscsiadm -m session					//检查是否有活动的session
$ iscsiadm -m node --logoutall=all		//注销所有session
$ iscsiadm -m node -T --logout				//选择一个目标来注销
$ iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799 -l       //报错，不能登录
$ iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799 -l -d8      //查看详细报错，-d是debug，8是调试级别。会发现没有使用认证
$ ls /var/lib/iscsi/nodes/				//有一个子目录。查看是否还有节点的定义，也就是discovery定义。可以直接从这个目录中删除，也可以使用以下命令
$ iscsiadm -m node -o delete 			//删除定义
$ ls /var/lib/iscsi/nodes/				//没有子目录了。同时iscsid.conf文件也应该被更新了。
$ iscsiadm -m discovery -t sendtargets -p 172.31.101.193        //再次执行discovery
$ iscsiadm -m node -T iqn.2003-01.org.linux-iscsi.server.x8664:sn.1cb9c232d799 -l       //登录。正常了
$ iscsiadm -m session					//检查是否有活动的session
```
# 4. 从LUKS加密文件系统恢复数据
LUKS(Linux Unified Key Setup)Linux统一密钥设置
Linux加密文件系统是通过LUKS来加密数据的
cryptsetup 设置、配置和管理LUKS
cryptsetup luksFormat 格式化为LUKS加密硬盘
cryptsetup luksOpen 访问被格式为LUKS的加密硬盘
cryptsetup luksDump 显示类似于dumpe2fs的信息
cryptsetup luksAddKey 添加密钥，以便能访问LUKS FS
cryptsetup luksRemoveKey 删除密钥
## 4.1 LUKS基础
pvcreate /dev/nvme1n1
vgcreate vg1 /dev/nvme1n1
lvcreate -L 1G -n luksvolume vg1
dnf install cryptsetup
cryptsetup luksFormat /dev/mapper/vg1-luksvolume //加密刚才创建的lv luksvolume。注意这会删除LV上所有数据，而且是不可逆的！
//回答YES
//添加 安装密码，这个密码用于打开LUKS加密卷
cryptsetup luksOpen /dev/mapper/vg1-luksvolume luks //访问这个LUKS卷，并将这个子卷命令为luks。在vg vg1下的lv luksvolume下生成一个LUKS luks
//输入访问密码
mkfs.xfs /dev/mapper/luks //在luks上创建xfs FS
mkidr /luks
mount -t xfs /dev/mapper/luks /luks/
echo 'hello' > /luks/hello
vi /etc/fstab
/dev/mapper/luks /luks xfs default 0 0
vi /etc/crypttab
luks /dev/mapper/vg1-luksvolume /root/secret.key
//luks的LV 映射到的LV 密钥引用的位置
cryptsetup luksDump /dev/mapper/vg1-luksvolume //显示luks的信息
//里面有 Keyslots，这里有我们的初始密码
vi /root/secret.key
AddaPassword //添加另外一个密码，是服务器在启动的时候用这个密码来访问加密数据
cryptsetup luksAddKey /dev/mapper/vg1-luksvolume /root/secret.key
//输入我们设置的初始口令
cryptsetup luksDump /dev/mapper/vg1-luksvolume //显示luks的信息
//里面有 Keyslots，这里多了一个1: luks2，也就是我们设置的key
## 4.2 luks头信息
### 4.2.1 备份luks头信息
cryptsetup luksHeaderBackup //备份luks头信息
cryptsetup luksHeaderBackup /dev/volume --header-backup-file /backup/location
cryptsetup luksHeaderBackup /dev/mapper/vg1-luksvolume --header-backup-file /root/luksbackup.header
### 4.2.2 恢复luks头信息
cryptsetup luksHeaderRestore //恢复luks头信息
cryptsetup luksHeaderRestore /dev/volume --header-backup-file /backup/location
cryptsetup luksHeaderRestore /dev/mapper/vg1-luksvolume --header-backup-file /root/luksbackup.header
### 4.2.3 例子
cryptsetup luksRemoveKey /dev/mapper/vg1-luksvolume --key-file /root/secret.key //破坏 密钥文件
umount /luks //卸载LUKS FS
cryptsetup luksClose /dev/mapper/luks //关闭LUKS FS
cryptsetup luksClose /dev/mapper/vg1-luksvolume //需要同时关闭它的LV
lvchange -an /dev/vg1/luksvolume //使之前的修改生效
lvchange -ay /dev/vg1/luksvolume
cryptsetup luksOpen /dev/mapper/vg1-luksvolume luks --key-file /root/secret.key //这里会报没有key文件的报
cryptsetup luksHeaderRestore /dev/mapper/vg1-luksvolume --header-backup-file /root/luksbackup.header //恢复LUKS头
//回答YES
cryptsetup luksOpen /dev/mapper/vg1-luksvolume luks --key-file /root/secret.key //一切正常
mount /luks/
ls /luks

