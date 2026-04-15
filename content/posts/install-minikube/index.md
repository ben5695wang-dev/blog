---
title: "安装minikube"
description: 
date: 2023-04-11T18:59:37+08:00
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
  - k8s

# 标签（Tags） - 适合具体关键词、多个细标签
tags:
  - CentOS
  - k8s
  - minikube
build:
    list: always    # Change to "never" to hide the page from the list
---

# CentOS7.9安装minikube
本文记录了在CentOS7.9上安装配置minikube的过程，目的是为了在单台机器上运行一个k8s的环境，而不需要创建一个k8s集群。


## 1.版本及安装前提
### 1.1版本概述
宿主机: Windows 10
虚拟机: VMware
虚拟机OS: CentOS7.9
MninKube: v1.30.1
Containerd: 1.6.20
Docker: 23.0.3
### 1.2安装前提
2 CPUs or more
2GB of free memory
20GB of free disk space
Internet connection
Container or virtual machine manager, such as: Docker, QEMU, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, or VMware Fusion/Workstation
link: https://minikube.sigs.k8s.io/docs/start
### 1.3过程概述
1. OS环境准备
2. 下载并安装CRI
3. 下载并安装minikube
4. 运行一个示例来测试minikube

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
### 2.4 将桥接的IPv4流量传递到iptables的链
```shell
# cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# sysctl --system
```
## 3.下载并安装CRI
### 3.1 安装docker

```shell
# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
......
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
## 4.下载并安装minikube
### 4.1 添加YUM软件源
先添加阿里云YUM软件源，不然没有yum源安装下面的软件包
```shell
# cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
### 4.2 安装kubectl
安装kubectl，会在minikube安装完成后使用；将用户加到docker组
```shell
# yum install -y kubectl
# usermod -aG docker $USER && newgrp docker
```
### 4.3 下载minikube
```shell
# curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
# install minikube-linux-amd64 /usr/local/bin/minikube
# chmod +x /usr/local/bin/minikube
```
### 4.4 运行安装minikube命令
```shell
# minikube start --force --driver=docker --cni calico  --registry-mirror=https://registry.docker-cn.com --container-runtime=containerd
```
## 5.运行一个示例来测试minikube
### 5.1 运行dashboard
```shell
# kubectl get pod -A
# minikube addons list
# minikube dashboard --url 
```
### 5.2 运行一个deploy来测试minikube
```shell
# kubectl create deployment nginx --image=nginx
# kubectl expose deployment nginx --port=80 --type=NodePort
# kubectl get svc
NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        5h13m
service/nginx        NodePort    10.105.88.53   <none>        80:30874/TCP   18s
# minikube service list
|----------------------|---------------------------|--------------|---------------------------|
|      NAMESPACE       |           NAME            | TARGET PORT  |            URL            |
|----------------------|---------------------------|--------------|---------------------------|
| default              | kubernetes                | No node port |                           |
| default              | nginx                     |           80 | http://192.168.49.2:32469 |
| kube-system          | kube-dns                  | No node port |                           |
| kubernetes-dashboard | dashboard-metrics-scraper | No node port |                           |
| kubernetes-dashboard | kubernetes-dashboard      | No node port |                           |
|----------------------|---------------------------|--------------|---------------------------|
```
通过 http://192.168.49.2:32469 可以正常访问nginx，代表minikube运行正常。