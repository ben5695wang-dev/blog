---
title: "安装配置私有仓库harbor"
description: 
date: 2023-04-10T18:38:59+08:00
image: 
math: 
license: 
comments: true
draft: false
toc: true
# 分类（Categories） - 适合大类、主要主题
categories:
  - 技术
  - CentOS
  - harbor

# 标签（Tags） - 适合具体关键词、多个细标签
tags:
  - CentOS
  - harbor
build:
    list: always    # Change to "never" to hide the page from the list
---

# 安装配置私有仓库Harbor
本文记录了在CentOS7.9上安装配置私有仓库Harbor的过程。

## 1.版本及安装前提
### 1.1版本概述
宿主机: Windows 10
### 1.2安装前提
硬件

| Resource   | Minimum    |  Recommended  |
| :---       | :----      | :----         |
| CPU        | 2cpu       |   4cpu        |
| Mem        | 4GB        |   8GB         |
| Disk       | 40GB       |   160GB       |

软件

| Software       | Version    |  Description  |
| ----           | :----      | :----         |
| Docker engine  | Version 17.06.0-ce+ or higher       |           |
| Docker Compose | docker-compose (v1.18.0+) or docker compose v2 (docker-compose-plugin)        |            |
| Openssl        | Latest is preferred       |          |

网络端口

| Port   | Protocol    |  Description |
| :---       | :----      | :----         |
| 443        | HTTPS       |   Harbor portal and core API accept HTTPS requests on this port. You can change this port in the configuration file.        |
| 443        | HTTPS        |   Connections to the Docker Content Trust service for Harbor. Only required if Notary is enabled. You can change this port in the configuration file.         |
| 80         | HTTP       |   Harbor portal and core API accept HTTP requests on this port. You can change this port in the configuration file.       |

## 2.OS环境准备
### 2.1 关闭防火墙：
```shell
# hostnamectl set-hostname minikube
# systemctl stop firewalld
# systemctl disable firewalld
```
### 2.2 关闭selinux
```shell
# sed -i 's/enforcing/disabled/' /etc/selinux/config
```
### 2.3 关闭swap
删除swap那一行，或者注释掉
```shell
# vim /etc/fstab 
```
### 2.4 挂载centos DVD，并作为本地yum源
在虚拟机上连接centos DVD
```shell
[root@harbor1 /]# cd /
[root@harbor1 /]# mkdir dvd0
[root@harbor1 /]# mount /dev/cdrom /dvd0
[root@harbor1 dvd0]# cd /etc/yum.repos.d/
[root@harbor1 yum.repos.d]# mkdir bak
[root@harbor1 yum.repos.d]# mv ./C* ./bak/
[root@harbor1 yum.repos.d]# vi centos7-local-Base.repo
[ISO]
name=redhat7.9-dvd
baseurl=file:///dvd0/
gpgcheck=0
enabled=1
[root@harbor1 yum.repos.d]# yum update
[root@harbor1 yum.repos.d]# yum repolist
```
### 2.4 下载需要的软件包，做自己的本地yum源
下载需要的软件包
```shell
[root@harbor1 yum.repos.d]# cp ./bak/* ./
[root@harbor1 dockce-local-repo]# yum install --downloadonly --downloaddir=/dockce-local-repo docker-ce
```
制作自己的yum源
```shell
[root@harbor1 /]# cd dockce-local-repo/
[root@harbor1 dockce-local-repo]# createrepo .
[root@harbor1 dockce-local-repo]# cd /etc/yum.repos.d/
[root@harbor1 yum.repos.d]# vi dockce-local.repo
[dockce-Base]
name=dockce local repo
baseurl=file:///dockce-local-repo
enabled=1
gpgcheck=0
priority=1
[root@harbor1 yum.repos.d]# yum update
[root@harbor1 yum.repos.d]# yum repolist
```

## 3.安装docker
### 3.1 安装docker

```shell
# yum -y install docker-ce
# systemctl enable docker && systemctl start docker
# docker version
```
### 3.2 配置镜像下载加速器
这里的地址使用的是个人的阿里云的镜像加速器
```shell
# cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://xxx.mirror.aliyuncs.com"]
}
EOF
# systemctl restart docker
```
### 3.3 安装Docker-Compose
使用场景：用来单机上编排容器(定义如何运行多个容器，使容器能互通)
docker compose是通过一个.yml/.yaml配置文件完成
docker compose将所管理的容器分为三层：
一个project包含多个service，一个service包含多个container
   工程/project  相当于一个目录，这个目录下有唯一的docker-compose.yml、extends文件和变量文件
         注意：如果没有指定project name就把目录名字作为project name
       服务/service  定义容器运行所需要的镜像、各种参数、依赖关系
            容器/container  数据运行的地方，最终运行的就是容器

从 https://github.com/docker/compose/releases 下载对应的包
然后上传到CentOS上
```shell
[root@harbor1 docker-compose-local]# cp ./docker-compose-linux-x86_64  /usr/local/bin/docker-compose
[root@harbor1 docker-compose-local]# chmod +x /usr/local/bin/docker-compose
[root@harbor1 docker-compose-local]# docker-compose version
```
## 4.安装Harbor
### 4.1下载Harbor离线安装包
从 https://github.com/goharbor/harbor/releases 下载对应的包
然后上传到CentOS上
### 4.2配置Harbor
```shell
[root@harbor1 harbor-local]# tar -zxf ./harbor-offline-installer-v2.8.0-rc2.tgz  -C /usr/local/
[root@harbor1 harbor]# cp harbor.yml.tmpl harbor.yml
[root@harbor1 harbor]# cp harbor.yml.tmpl harbor.yml
[root@harbor1 harbor]# vim harbor.yml
hostname: 192.168.50.160        //Harbor的域名或者IP

# http related config
http:       
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80      //HTTP端口

# https related config
#https:     //HTTPS相关，如果不对外服务，可以并掉
#  # https port for harbor, default is 443
#  port: 443    //如果设置了HTTPS端口，那么HTTP端口会重定向到HTTPS端口中
#  # The path of cert and key files for nginx
#  certificate: /your/certificate/path      //证书路径
#  private_key: /your/private/key/path      //证书路径
......
# The initial password of Harbor admin
# It only works in first time to install harbor
# Remember Change the admin password from UI after launching Harbor.
harbor_admin_password: Harbor12345      //admin账户密码

# Harbor DB configuration
database:
  # The password for the root user of Harbor DB. Change this before any production use.
  password: root123          //数据库密码
......
# The default data volume
data_volume: /Harbordata                //数据库存放地址
```
### 4.3 安装Harbor
```shell
[root@harbor1 harbor]# ./install.sh
```
安装完成后，就可以通过配置文件中的IP/域名访问，用户名为admin，密码在配置文件中。
### 4.4查看Harbor
要在/usr/local/harbor这个目录下，因为这个目录下有docker-compose.yaml这个文件，不然会报以下错误
docker-compose启动容器报“no configuration file provided: not found”，配置文件未找到。其中最常见的原因是没有在有docker-compose.yaml的路径下执行该命令。
```shell
[root@harbor1 harbor]# cd /usr/local/harbor
[root@harbor1 harbor]# docker-compose ps 
```
### 4.5重启Harbor
```shell
[root@harbor1 harbor]# cd /usr/local/harbor
[root@harbor1 harbor]# docker-compose stop
[root@harbor1 harbor]# docker-compose up -d
[root@harbor1 harbor]# docker-compose ps | wc
     10     127    1655
```
## 5.使用Harbor
管理员用户拥有创建仓库和删除镜像的权限 
普通用户拥有上传镜像和拉取镜像权限 无法删除仓库和镜像
### 5.1创建项目
公开项目：所有用户都快要访问，通常存放公共的镜像，默认有一个library公开项目
私有项目：只有授权用户才可以访问，通换成那个存放项目本身的镜像
然后可以创建用户，并分配项目权限,之后上传镜像可以登录创建的用户
创建一个公开项目 k8s
### 5.2创建用户
使用admin登录用，创建一个账户
### 5.3在项目中添加用户
在创建的公开项目中，选择成员，添加成员，把新创建的用户k8s添加进去 
### 5.4配置docker
找到docker 的 daemon.json 配置文件，CentOS 7 的路径：/etc/docker/daemon.json（系统版本不同所处位置不同，其他版本的请自行百度），如果路径下没有这个文件自己创建即可。然后再配置文件里加上：
这里要添加不安全的网址和端口号，不会login的时候会直接去找443端口而报错。
```shell
[root@harbor1 docker]# vi /etc/docker/daemon.json
{
  "insecure-registries": ["192.168.50.160:80"]
}
[root@harbor1 docker]# systemctl daemon-reload
[root@harbor1 docker]# systemctl restart docker.service
```
### 5.5手动下载image
1.在可以联网的机器上面下载需要的镜像，传到可以联到Harbor的机器上面
grep image calico.yaml //查看都需要拉取哪些镜像，然后
```shell
[root@harbor1 ~]# docker pull docker.io/calico/cni:v3.22.1 
[root@harbor1 harbor]# cd /
[root@harbor1 /]# mkdir docker-local-images
[root@harbor1 /]# cd docker-local-images/
[root@harbor1 docker-local-images]# docker save calico/cni:v3.22.1 > cni.tar 
[root@harbor1 docker-local-images]# scp /docker-local-images/cni.tar root@k8s-node1:/docker-local-images/cni.tar
```

2.在可以联到Harbor的机器上面
tag里面要添加端口号，不然push的时候会直接去找443端口，从而报错。
login的时候要指明端口号，不然lgoin的时候会直接去找443端口，从而报错。
```shell
[root@harbor1 harbor]# docker load < /docker-local-images/cni.tar	//从别的机器上拷贝过来，然后导入tar文件
[root@harbor1 harbor]# docker tag calico/cni:v3.22.1 192.168.50.160:80/k8s/cni:v3.22.1
[root@harbor1 harbor]# docker login 192.168.50.160:80   //使用上面创建的k8s用户名登录
[root@harbor1 harbor]# docker push 192.168.50.160:80/k8s/cni:v3.22.1
```

## 6.下载k8s安装需要的内容
### 6.1下载kubelet kubeadm kubectl
```shell
[root@harbor1 kubelet]# yum install --downloadonly --downloaddir=/dockce-local-repo/kubelet kubelet kubeadm kubectl
[root@harbor1 yum.repos.d]# cd /dockce-local-repo/
[root@harbor1 dockce-local-repo]# createrepo .
[root@harbor1 yum.repos.d]# yum install -y kubelet kubeadm kubectl
```
### 6.2下载kubadm init需要的docker image
```shell
[root@harbor1 yum.repos.d]# kubeadm config images list
W0416 00:36:46.142279  106359 images.go:80] could not find officially supported version of etcd for Kubernetes v1.27.1, falling back to the nearest etcd version (3.5.7-0)
registry.k8s.io/kube-apiserver:v1.27.1
registry.k8s.io/kube-controller-manager:v1.27.1
registry.k8s.io/kube-scheduler:v1.27.1
registry.k8s.io/kube-proxy:v1.27.1
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.7-0
registry.k8s.io/coredns/coredns:v1.10.1
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.27.1
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.27.1
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.27.1
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.27.1
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.7-0
[root@harbor1 yum.repos.d]# docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.10.1
```
将registry.k8s.io替换为registry.cn-hangzhou.aliyuncs.com/google_containers，然后拉取
registry.k8s.io/coredns/要替换为egistry.cn-hangzhou.aliyuncs.com/google_containers，没有/coredns/,然后拉取