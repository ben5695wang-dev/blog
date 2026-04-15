---
title: "搭建不需要外网的局域网内yum源"
description: 
date: 2023-04-02T19:31:55+08:00
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
  - yum
# 标签（Tags） - 适合具体关键词、多个细标签
tags:
  - Linux
  - yum
build:
    list: always    # Change to "never" to hide the page from the list
---

# 搭建不需要外网的局域网内yum源
将要安装的包，通过能上网的机器下载下来，然后copy到离线的Linux上，创建不需要外网的局域网内yum源，就可以进行局域网内安装了。
## 1. yum安装过程
yum安装的本质是把后缀名.rpm的包下载到本地，然后按次序安装，但是每次yum install xxx时，都会自动安装并且在安装完成直接后把rpm包自动删除。
当我们下载比较大或者网络不好的时候，每次都需要重新下载就会非常慢；或者需要重复安装多台机器或重复安装多次的时候，在这种情况下就可以考虑搭建不需要外网的局域网内yum源仓库。

搭建不需要外网的局域网内yum源仓库的几个步骤：
1. 将rpm包下载到本地。
2. 生成repodata信息。
3. 安装配置http服务。
4. 创建对应yum源文件。
5. 使用yum repoinfo检查确认。
6. 对本地yum源仓库进行更新。

## 2. 搭建不需要外网的局域网内yum源仓库
### 2.1 将rpm包下载到本地
创建一个本地文件夹用于存放要下载的安装包，然后用yum命令的downloadonly选项来下载要安装的包及其依赖包。在能上网的Linux上执行以下命令：
```shell
[root@harbor1 /]# cd /
[root@harbor1 /]# mkdir whxlocalyum
[root@harbor1 /]# yum install  --downloadonly --downloaddir=/whxlocalyum httpd
```
### 2.2 生成repodata信息
将以上内容copy到离线机器中的某个文件夹中，这里用/whxlocalyum文件夹。然后使用createrepo命令生成repodata信息。

```shell
[root@harbor1 whxlocalyum]# yum install createrepo -y
[root@harbor1 whxlocalyum]# tree /whxlocalyum/
[root@harbor1 whxlocalyum]# createrepo /whxlocalyum/
```
### 2.3 安装配置http服务
这里使用了docker来运行http服务
这里是将宿主机的81端口映射到了docker容器httpd的80端口，将宿主机的/whxlocalyum文件夹映射到了docker容器httpd的/usr/local/apache2/htdocs/whxlocalnetworkyum文件夹
```shell
[root@harbor1 /]# docker pull httpd
[root@harbor1 /]# docker run -d -p 81:80 -v /whxlocalyum:/usr/local/apache2/htdocs/whxlocalnetworkyum httpd:latest 
e25750f8aca0362db4a9aa4521518facace669128ab69f9c37a41352c7dd19e1
[root@harbor1 /]# docker exec -it e25750f8aca0 bash
root@e25750f8aca0:/usr/local/apache2# cd htdocs/
root@e25750f8aca0:/usr/local/apache2/htdocs# ls -al
```
如果docker中没有vi，可以先在docker里面安装vi:
需要先用root用户进入 docker
```shell
[root@harbor1 whxlocalyum]# docker exec -it --user root 9f7784560008 bash
root@9f7784560008:/usr/local/apache2# apt-get update
root@9f7784560008:/usr/local/apache2# apt-get install vim
```
### 2.4 创建对应yum源文件
手动在/etc/yum.repos.d/目录下创建局域网yum源文件whxlocalnetworkyum.repo
```shell
[root@harbor1 yum.repos.d]# vi whxlocalnetworkyum.repo 
[whxlocalnetworkyum]
name=whxlocalnetworkyum
baseurl=http://192.168.1.160:81/whxlocalnetworkyum
gpgcheck=0
enabled=1
~
```
### 2.5 使用yum repoinfo检查确认
```shell
[root@harbor1 yum.repos.d]# yum repolist
[root@harbor1 yum.repos.d]# yum repoinfo whxlocalnetworkyum
```
### 2.5 对本地yum源仓库进行更新
每次下载一个新的rpm软件包到本地仓库后，我们使用yum repoinfo whxcrio查看会发现软件包的数量并没有增加，我们安装新增的软件包也会提示找不到这个软件包，这是因为新下载的rpm包没有更新到仓库信息中。执行以下命令对本地仓库进行更新：
   1. 查看旧的软件包总数 yum repoinfo whxcrio | grep pkgs
   2. 更新本地仓库 createrepo --update /whxlocalyum/
   3. 清除所有缓存 yum clean all
   4. 查看新的软件包总数 yum repoinfo whxcrio | grep pkgs

```shell
[root@harbor1 whxlocalyum]# yum install  --downloadonly --downloaddir=/whxlocalyum vsftpd
[root@harbor1 yum.repos.d]# yum repoinfo whxlocalnetworkyum
[root@harbor1 yum.repos.d]#  yum repoinfo whxlocalnetworkyum  | grep pkgs
[root@harbor1 whxlocalyum]# createrepo --update /whxlocalyum/
[root@harbor1 whxlocalyum]# yum  clean all
[root@harbor1 whxlocalyum]#  yum repoinfo whxlocalnetworkyum  | grep pkgs
```
## 3. 使用局域网内yum源
[^_^]: 可以看到，还是有一些是从其他仓库安装的，要注意？
```shell
[root@harbor1 whxlocalyum]# cd /etc/yum.repos.d/
[root@harbor1 yum.repos.d]# ls -al
[root@harbor1 yum.repos.d]# cat whxlocalnetworkyum.repo 
[whxlocalnetworkyum]
name=whxlocalnetworkyum
baseurl=http://192.168.50.160:81/whxlocalnetworkyum
gpgcheck=0
enabled=1
[root@harbor1 yum.repos.d]# yum install vsftpd
```
## 4.相关log全过程
[搭建不需要外网的局域网内yum源_log](http://39.105.160.122/2023/04/02/%e6%90%ad%e5%bb%ba%e4%b8%8d%e9%9c%80%e8%a6%81%e5%a4%96%e7%bd%91%e7%9a%84%e5%b1%80%e5%9f%9f%e7%bd%91%e5%86%85yum%e6%ba%90_log/ "搭建不需要外网的局域网内yum源_log")
