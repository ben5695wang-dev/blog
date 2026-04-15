---
title: "搭建不需要外网的局域网内yum源_log"
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
  - log
build:
    list: always    # Change to "never" to hide the page from the list
---

## 2. 搭建不需要外网的局域网内yum源仓库
### 2.1 将rpm包下载到本地
```shell
[root@harbor1 /]# cd /
[root@harbor1 /]# mkdir whxlocalyum
[root@harbor1 /]# yum install  --downloadonly --downloaddir=/whxlocalyum httpd
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
file:///dvd0/repodata/repomd.xml: [Errno 14] curl#37 - "Couldn't open file /dvd0/repodata/repomd.xml"
Trying other mirror.
base                                                                                                                                                                                                                  | 3.6 kB  00:00:00     
dockce-Base                                                                                                                                                                                                           | 2.9 kB  00:00:00     
docker-ce-stable                                                                                                                                                                                                      | 3.5 kB  00:00:00     
extras                                                                                                                                                                                                                | 2.9 kB  00:00:00     
kubernetes                                                                                                                                                                                                            | 1.4 kB  00:00:00     
updates                                                                                                                                                                                                               | 2.9 kB  00:00:00     
Resolving Dependencies
--> Running transaction check
---> Package httpd.x86_64 0:2.4.6-98.el7.centos.7 will be installed
--> Processing Dependency: httpd-tools = 2.4.6-98.el7.centos.7 for package: httpd-2.4.6-98.el7.centos.7.x86_64
--> Processing Dependency: /etc/mime.types for package: httpd-2.4.6-98.el7.centos.7.x86_64
--> Processing Dependency: libaprutil-1.so.0()(64bit) for package: httpd-2.4.6-98.el7.centos.7.x86_64
--> Processing Dependency: libapr-1.so.0()(64bit) for package: httpd-2.4.6-98.el7.centos.7.x86_64
--> Running transaction check
---> Package apr.x86_64 0:1.4.8-7.el7 will be installed
---> Package apr-util.x86_64 0:1.5.2-6.el7 will be installed
---> Package httpd-tools.x86_64 0:2.4.6-98.el7.centos.7 will be installed
---> Package mailcap.noarch 0:2.1.41-2.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================================================================================================================================
 Package                                                  Arch                                                Version                                                             Repository                                            Size
=============================================================================================================================================================================================================================================
Installing:
 httpd                                                    x86_64                                              2.4.6-98.el7.centos.7                                               updates                                              2.7 M
Installing for dependencies:
 apr                                                      x86_64                                              1.4.8-7.el7                                                         ISO                                                  104 k
 apr-util                                                 x86_64                                              1.5.2-6.el7                                                         ISO                                                   92 k
 httpd-tools                                              x86_64                                              2.4.6-98.el7.centos.7                                               updates                                               94 k
 mailcap                                                  noarch                                              2.1.41-2.el7                                                        ISO                                                   31 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package (+4 Dependent packages)

Total download size: 3.0 M
Installed size: 10 M
Background downloading packages, then exiting:
(1/2): httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm                                                                                                                                                                   |  94 kB  00:00:00     
(2/2): httpd-2.4.6-98.el7.centos.7.x86_64.rpm                                                                                                                                                                         | 2.7 MB  00:00:01     


Error downloading packages:
  apr-util-1.5.2-6.el7.x86_64: [Errno 256] No more mirrors to try.
  apr-1.4.8-7.el7.x86_64: [Errno 256] No more mirrors to try.
  mailcap-2.1.41-2.el7.noarch: [Errno 256] No more mirrors to try.

[root@harbor1 /]# 
[root@harbor1 /]# cd /whxlocalyum/
[root@harbor1 whxlocalyum]# ls -al
total 2884
drwxr-xr-x   2 root root     104 Apr 24 23:37 .
dr-xr-xr-x. 24 root root    4096 Apr 24 23:36 ..
-rw-r--r--   1 root root 2849180 Apr  5 13:37 httpd-2.4.6-98.el7.centos.7.x86_64.rpm
-rw-r--r--   1 root root   96652 Apr  5 13:37 httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm
```
### 2.2 生成repodata信息
```shell
[root@harbor1 whxlocalyum]# yum install createrepo -y
[root@harbor1 whxlocalyum]# tree /whxlocalyum/
bash: tree: command not found...
[root@harbor1 yum.repos.d]# cd /
[root@harbor1 /]# tree
bash: tree: command not found...
[root@harbor1 /]# yum install tree
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package tree.x86_64 0:1.6.0-10.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================================================================================================================================
 Package                                                Arch                                                     Version                                                        Repository                                              Size
=============================================================================================================================================================================================================================================
Installing:
 tree                                                   x86_64                                                   1.6.0-10.el7                                                   base                                                    46 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package

Total download size: 46 k
Installed size: 87 k
Is this ok [y/d/N]: y
Downloading packages:
tree-1.6.0-10.el7.x86_64.rpm                                                                                                                                                                                          |  46 kB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : tree-1.6.0-10.el7.x86_64                                                                                                                                                                                                  1/1 
  Verifying  : tree-1.6.0-10.el7.x86_64                                                                                                                                                                                                  1/1 

Installed:
  tree.x86_64 0:1.6.0-10.el7                                                                                                                                                                                                                 

Complete!
[root@harbor1 whxlocalyum]# tree /whxlocalyum/
/whxlocalyum/
|-- httpd-2.4.6-98.el7.centos.7.x86_64.rpm
`-- httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm

0 directories, 2 files
[root@harbor1 /]# cd /whxlocalyum/
[root@harbor1 whxlocalyum]# cd /
[root@harbor1 /]# createrepo /whxlocalyum/
Spawning worker 0 with 1 pkgs
Spawning worker 1 with 1 pkgs
Spawning worker 2 with 0 pkgs
Spawning worker 3 with 0 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
[root@harbor1 /]# cd /whxlocalyum/
[root@harbor1 whxlocalyum]# ls -al
total 2888
drwxr-xr-x   3 root root     120 Apr 25 00:07 .
dr-xr-xr-x. 24 root root    4096 Apr 24 23:36 ..
-rw-r--r--   1 root root 2849180 Apr  5 13:37 httpd-2.4.6-98.el7.centos.7.x86_64.rpm
-rw-r--r--   1 root root   96652 Apr  5 13:37 httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm
drwxr-xr-x   2 root root    4096 Apr 25 00:07 repodata
[root@harbor1 whxlocalyum]# 
```

### 2.3 安装配置http服务
```shell
[root@harbor1 /]# docker image ls
REPOSITORY                                                                    TAG       IMAGE ID       CREATED        SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver            v1.27.1   6f6e73fa8162   10 days ago    121MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager   v1.27.1   c6b511817822   10 days ago    112MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler            v1.27.1   6468fa8f9869   10 days ago    58.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy                v1.27.1   fbe39e5d66b6   10 days ago    71.1MB
goharbor/harbor-exporter                                                      v2.8.0    2abc02438fcf   11 days ago    97.1MB
goharbor/redis-photon                                                         v2.8.0    ef1f410f9255   11 days ago    127MB
goharbor/trivy-adapter-photon                                                 v2.8.0    724824e3559a   11 days ago    454MB
goharbor/notary-server-photon                                                 v2.8.0    d603449fe91f   11 days ago    113MB
goharbor/notary-signer-photon                                                 v2.8.0    618fc02c41bf   11 days ago    110MB
goharbor/harbor-registryctl                                                   v2.8.0    165749c6eedc   11 days ago    141MB
goharbor/registry-photon                                                      v2.8.0    8bfd12c2163d   11 days ago    78.5MB
goharbor/nginx-photon                                                         v2.8.0    cfc2401896e1   11 days ago    126MB
goharbor/harbor-log                                                           v2.8.0    f31ccc3d46f0   11 days ago    134MB
goharbor/harbor-jobservice                                                    v2.8.0    1b00a3a474e1   11 days ago    140MB
goharbor/harbor-core                                                          v2.8.0    15f4066c1707   11 days ago    164MB
goharbor/harbor-portal                                                        v2.8.0    ae18a071cdce   11 days ago    133MB
goharbor/harbor-db                                                            v2.8.0    f3d4373617a2   11 days ago    179MB
goharbor/prepare                                                              v2.8.0    daa44ccf3b06   11 days ago    170MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                   v1.10.1   ead0a4a53df8   2 months ago   53.6MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd                      3.5.7-0   86b6af7dd652   2 months ago   296MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                     3.9       e6f181688397   6 months ago   744kB
[root@harbor1 /]# docker pull httpd
Using default tag: latest
latest: Pulling from library/httpd
26c5c85e47da: Pull complete 
2d29d3837df5: Pull complete 
2483414a5e59: Pull complete 
e78016c4ba87: Pull complete 
757908175415: Pull complete 
Digest: sha256:a182ef2350699f04b8f8e736747104eb273e255e818cd55b6d7aa50a1490ed0c
Status: Downloaded newer image for httpd:latest
docker.io/library/httpd:latest
[root@harbor1 /]# docker image ls
REPOSITORY                                                                    TAG       IMAGE ID       CREATED        SIZE
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver            v1.27.1   6f6e73fa8162   10 days ago    121MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler            v1.27.1   6468fa8f9869   10 days ago    58.4MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager   v1.27.1   c6b511817822   10 days ago    112MB
registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy                v1.27.1   fbe39e5d66b6   10 days ago    71.1MB
goharbor/harbor-exporter                                                      v2.8.0    2abc02438fcf   11 days ago    97.1MB
goharbor/redis-photon                                                         v2.8.0    ef1f410f9255   11 days ago    127MB
goharbor/trivy-adapter-photon                                                 v2.8.0    724824e3559a   11 days ago    454MB
goharbor/notary-server-photon                                                 v2.8.0    d603449fe91f   11 days ago    113MB
goharbor/notary-signer-photon                                                 v2.8.0    618fc02c41bf   11 days ago    110MB
goharbor/harbor-registryctl                                                   v2.8.0    165749c6eedc   11 days ago    141MB
goharbor/registry-photon                                                      v2.8.0    8bfd12c2163d   11 days ago    78.5MB
goharbor/nginx-photon                                                         v2.8.0    cfc2401896e1   11 days ago    126MB
goharbor/harbor-log                                                           v2.8.0    f31ccc3d46f0   11 days ago    134MB
goharbor/harbor-jobservice                                                    v2.8.0    1b00a3a474e1   11 days ago    140MB
goharbor/harbor-core                                                          v2.8.0    15f4066c1707   11 days ago    164MB
goharbor/harbor-portal                                                        v2.8.0    ae18a071cdce   11 days ago    133MB
goharbor/harbor-db                                                            v2.8.0    f3d4373617a2   11 days ago    179MB
goharbor/prepare                                                              v2.8.0    daa44ccf3b06   11 days ago    170MB
httpd                                                                         latest    4b7fc736cb48   13 days ago    145MB
registry.cn-hangzhou.aliyuncs.com/google_containers/coredns                   v1.10.1   ead0a4a53df8   2 months ago   53.6MB
registry.cn-hangzhou.aliyuncs.com/google_containers/etcd                      3.5.7-0   86b6af7dd652   2 months ago   296MB
registry.cn-hangzhou.aliyuncs.com/google_containers/pause                     3.9       e6f181688397   6 months ago   744kB
[root@harbor1 /]# 
[root@harbor1 whxlocalyum]# docker run -d -p 81:80 -v /whxlocalyum:/usr/local/apache2/htdocs/whxlocalnetworkyum httpd:latest 
e25750f8aca0362db4a9aa4521518facace669128ab69f9c37a41352c7dd19e1
[root@harbor1 /]# docker ps
CONTAINER ID   IMAGE          COMMAND              CREATED         STATUS         PORTS                               NAMES
e25750f8aca0   httpd:latest   "httpd-foreground"   2 minutes ago   Up 2 minutes   0.0.0.0:81->80/tcp, :::81->80/tcp   blissful_varahamihira
[root@harbor1 /]# docker ps -a
CONTAINER ID   IMAGE                                COMMAND                  CREATED         STATUS                     PORTS                               NAMES
e25750f8aca0   httpd:latest                         "httpd-foreground"       2 minutes ago   Up 2 minutes               0.0.0.0:81->80/tcp, :::81->80/tcp   blissful_varahamihira
9e01b9a9825f   goharbor/harbor-jobservice:v2.8.0    "/harbor/entrypoint.鈥?   9 days ago      Exited (2) 2 hours ago                                         harbor-jobservice
cb52440e421d   goharbor/nginx-photon:v2.8.0         "nginx -g 'daemon of鈥?   9 days ago      Exited (0) 2 hours ago                                         nginx
f8bb7a6ac17b   goharbor/harbor-core:v2.8.0          "/harbor/entrypoint.鈥?   9 days ago      Exited (2) 2 hours ago                                         harbor-core
4ebab6a511df   goharbor/harbor-db:v2.8.0            "/docker-entrypoint.鈥?   9 days ago      Exited (0) 2 hours ago                                         harbor-db
b8da2fcbb185   goharbor/redis-photon:v2.8.0         "redis-server /etc/r鈥?   9 days ago      Exited (0) 2 hours ago                                         redis
07ef64965450   goharbor/registry-photon:v2.8.0      "/home/harbor/entryp鈥?   9 days ago      Exited (2) 2 hours ago                                         registry
6932a15aa6f9   goharbor/harbor-portal:v2.8.0        "nginx -g 'daemon of鈥?   9 days ago      Exited (0) 2 hours ago                                         harbor-portal
cf1948a25f0d   goharbor/harbor-registryctl:v2.8.0   "/home/harbor/start.鈥?   9 days ago      Exited (137) 2 hours ago                                       registryctl
0f04a60bc74c   goharbor/harbor-log:v2.8.0           "/bin/sh -c /usr/loc鈥?   9 days ago      Exited (137) 2 hours ago                                       harbor-log
[root@harbor1 /]# docker exec -it e25750f8aca0 bash
root@e25750f8aca0:/usr/local/apache2# ls 
bin  build  cgi-bin  conf  error  htdocs  icons  include  logs  modules
root@e25750f8aca0:/usr/local/apache2# cd htdocs/
root@e25750f8aca0:/usr/local/apache2/htdocs# ls -al
total 4
drwxr-xr-x 1 root     root      32 Apr 25 04:42 .
drwxr-xr-x 1 www-data www-data  32 Apr 12 01:47 ..
-rw-r--r-- 1      501 staff     45 Jun 11  2007 index.html
drwxrwxrwx 3 root     root     120 Apr 25 04:07 whxlocalnetworkyum
root@e25750f8aca0:/usr/local/apache2/htdocs# cd whxlocalnetworkyum/
root@e25750f8aca0:/usr/local/apache2/htdocs/whxlocalnetworkyum# ls -al
total 2884
drwxrwxrwx 3 root root     120 Apr 25 04:07 .
drwxr-xr-x 1 root root      32 Apr 25 04:42 ..
-rwxrwxrwx 1 root root 2849180 Apr  5 17:37 httpd-2.4.6-98.el7.centos.7.x86_64.rpm
-rwxrwxrwx 1 root root   96652 Apr  5 17:37 httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm
drwxrwxrwx 2 root root    4096 Apr 25 04:07 repodata
root@e25750f8aca0:/usr/local/apache2/htdocs/whxlocalnetworkyum# 
root@e25750f8aca0:/usr/local/apache2/htdocs/whxlocalnetworkyum# exit
```

### 2.4 创建对应yum源文件，使用yum repoinfo检查确认
```shell
[root@harbor1 /]# cd /etc/yum.repos.d
[root@harbor1 yum.repos.d]# vi whxlocalnetworkyum.repo

[whxlocalnetworkyum]
name=whxlocalnetworkyum
baseurl=http://192.168.1.160:81/whxlocalnetworkyum
gpgcheck=0
enabled=1
~
[root@harbor1 yum.repos.d]# mv ./* ./bak/
[root@harbor1 yum.repos.d]# ls -al
total 16
drwxr-xr-x.   3 root root   17 Apr 25 00:49 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
drwxr-xr-x    2 root root 4096 Apr 25 00:49 bak
[root@harbor1 yum.repos.d]# mv ./bak/whxlocalnetworkyum.repo .
[root@harbor1 yum.repos.d]# ls -al
total 20
drwxr-xr-x.   3 root root   48 Apr 25 00:49 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
drwxr-xr-x    2 root root 4096 Apr 25 00:49 bak
-rw-r--r--    1 root root  117 Apr 25 00:48 whxlocalnetworkyum.repo
[root@harbor1 yum.repos.d]# 
[root@harbor1 yum.repos.d]# pwd
/etc/yum.repos.d
[root@harbor1 yum.repos.d]# ls -al
total 20
drwxr-xr-x.   3 root root   48 Apr 25 00:49 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
drwxr-xr-x    2 root root 4096 Apr 25 00:49 bak
-rw-r--r--    1 root root  117 Apr 25 00:48 whxlocalnetworkyum.repo
[root@harbor1 yum.repos.d]# cat whxlocalnetworkyum.repo 
[whxlocalnetworkyum]
name=whxlocalnetworkyum
baseurl=http://192.168.1.160:81/whxlocalnetworkyum
gpgcheck=0
enabled=1
[root@harbor1 yum.repos.d]# 
[root@harbor1 yum.repos.d]# yum repolist
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
repo id                                                                                                            repo name                                                                                                           status
whxlocalnetworkyum                                                                                                 whxlocalnetworkyum                                                                                                  2
repolist: 2
[root@harbor1 yum.repos.d]# yum repoinfo whxlocalnetworkyum
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Repo-id      : whxlocalnetworkyum
Repo-name    : whxlocalnetworkyum
Repo-status  : enabled
Repo-revision: 1682395624
Repo-updated : Tue Apr 25 00:07:04 2023
Repo-pkgs    : 2
Repo-size    : 2.8 M
Repo-baseurl : http://192.168.50.160:81/whxlocalnetworkyum/
Repo-expire  : 21600 second(s) (last: Tue Apr 25 01:17:13 2023)
  Filter     : read-only:present
Repo-filename: /etc/yum.repos.d/whxlocalnetworkyum.repo

repolist: 2
[root@harbor1 yum.repos.d]# 
```
### 2.5 对本地yum源仓库进行更新
```shell
[root@harbor1 yum.repos.d]# cp ./bak/* .
[root@harbor1 yum.repos.d]# ls -al
total 84
drwxr-xr-x.   3 root root 4096 Apr 25 01:20 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
-rw-r--r--    1 root root 2523 Apr 25 01:20 CentOS-Base.repo
-rw-r--r--    1 root root 1664 Apr 25 01:20 CentOS-Base.repo.backup
-rw-r--r--    1 root root 1309 Apr 25 01:20 CentOS-CR.repo
-rw-r--r--    1 root root  649 Apr 25 01:20 CentOS-Debuginfo.repo
-rw-r--r--    1 root root  630 Apr 25 01:20 CentOS-Media.repo
-rw-r--r--    1 root root 1331 Apr 25 01:20 CentOS-Sources.repo
-rw-r--r--    1 root root 8515 Apr 25 01:20 CentOS-Vault.repo
-rw-r--r--    1 root root  314 Apr 25 01:20 CentOS-fasttrack.repo
-rw-r--r--    1 root root  616 Apr 25 01:20 CentOS-x86_64-kernel.repo
drwxr-xr-x    2 root root 4096 Apr 25 00:49 bak
-rw-r--r--    1 root root  103 Apr 25 01:20 dockce-local.repo
-rw-r--r--    1 root root 2081 Apr 25 01:20 docker-ce.repo
-rw-r--r--    1 root root  275 Apr 25 01:20 kubernetes.repo
-rw-r--r--    1 root root  118 Apr 25 01:17 whxlocalnetworkyum.repo
[root@harbor1 whxlocalyum]# yum install  --downloadonly --downloaddir=/whxlocalyum vsftpd
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
Resolving Dependencies
--> Running transaction check
---> Package vsftpd.x86_64 0:3.0.2-29.el7_9 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================================================================================================================================
 Package                                                Arch                                                   Version                                                         Repository                                               Size
=============================================================================================================================================================================================================================================
Installing:
 vsftpd                                                 x86_64                                                 3.0.2-29.el7_9                                                  updates                                                 173 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package

Total download size: 173 k
Installed size: 353 k
Background downloading packages, then exiting:
vsftpd-3.0.2-29.el7_9.x86_64.rpm                                                                                                                                                                                      | 173 kB  00:00:00     
exiting because "Download Only" specified
[root@harbor1 whxlocalyum]# ls -al
total 3064
drwxrwxrwx   3 root root     160 Apr 25 01:23 .
dr-xr-xr-x. 24 root root    4096 Apr 24 23:36 ..
-rwxrwxrwx   1 root root 2849180 Apr  5 13:37 httpd-2.4.6-98.el7.centos.7.x86_64.rpm
-rwxrwxrwx   1 root root   96652 Apr  5 13:37 httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm
drwxrwxrwx   2 root root    4096 Apr 25 00:07 repodata
-rw-r--r--   1 root root  176896 Jun 11  2021 vsftpd-3.0.2-29.el7_9.x86_64.rpm
[root@harbor1 whxlocalyum]# cd /etc/yum.repos.d/
[root@harbor1 yum.repos.d]# ls -al
total 80
drwxr-xr-x.   3 root root 4096 Apr 25 01:21 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
-rw-r--r--    1 root root 2523 Apr 25 01:20 CentOS-Base.repo
-rw-r--r--    1 root root 1664 Apr 25 01:20 CentOS-Base.repo.backup
-rw-r--r--    1 root root 1309 Apr 25 01:20 CentOS-CR.repo
-rw-r--r--    1 root root  649 Apr 25 01:20 CentOS-Debuginfo.repo
-rw-r--r--    1 root root  630 Apr 25 01:20 CentOS-Media.repo
-rw-r--r--    1 root root 1331 Apr 25 01:20 CentOS-Sources.repo
-rw-r--r--    1 root root 8515 Apr 25 01:20 CentOS-Vault.repo
-rw-r--r--    1 root root  314 Apr 25 01:20 CentOS-fasttrack.repo
-rw-r--r--    1 root root  616 Apr 25 01:20 CentOS-x86_64-kernel.repo
drwxr-xr-x    2 root root 4096 Apr 25 01:21 bak
-rw-r--r--    1 root root  103 Apr 25 01:20 dockce-local.repo
-rw-r--r--    1 root root 2081 Apr 25 01:20 docker-ce.repo
-rw-r--r--    1 root root  275 Apr 25 01:20 kubernetes.repo
-rw-r--r--    1 root root  118 Apr 25 01:17 whxlocalnetworkyum.repo
[root@harbor1 yum.repos.d]# mv ./* ./bak/
mv: cannot move './bak' to a subdirectory of itself, './bak/bak'
mv: overwrite './bak/CentOS-Base.repo'? y
mv: overwrite './bak/CentOS-Base.repo.backup'? y
mv: overwrite './bak/CentOS-CR.repo'? y
mv: overwrite './bak/CentOS-Debuginfo.repo'? y
mv: overwrite './bak/CentOS-fasttrack.repo'? y
mv: overwrite './bak/CentOS-Media.repo'? y
mv: overwrite './bak/CentOS-Sources.repo'? y
mv: overwrite './bak/CentOS-Vault.repo'? y
mv: overwrite './bak/CentOS-x86_64-kernel.repo'? y
mv: overwrite './bak/dockce-local.repo'? y
mv: overwrite './bak/docker-ce.repo'? y
mv: overwrite './bak/kubernetes.repo'? y
[root@harbor1 yum.repos.d]# ls -al
total 16
drwxr-xr-x.   3 root root   17 Apr 25 01:25 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
drwxr-xr-x    2 root root 4096 Apr 25 01:25 bak
[root@harbor1 yum.repos.d]# mv ./bak/whxlocalnetworkyum.repo .
[root@harbor1 yum.repos.d]# ls -al
total 20
drwxr-xr-x.   3 root root   48 Apr 25 01:26 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
drwxr-xr-x    2 root root 4096 Apr 25 01:26 bak
-rw-r--r--    1 root root  118 Apr 25 01:17 whxlocalnetworkyum.repo
[root@harbor1 yum.repos.d]# 
[root@harbor1 yum.repos.d]# yum repoinfo whxlocalnetworkyum
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Repo-id      : whxlocalnetworkyum
Repo-name    : whxlocalnetworkyum
Repo-status  : enabled
Repo-revision: 1682395624
Repo-updated : Tue Apr 25 00:07:04 2023
Repo-pkgs    : 2
Repo-size    : 2.8 M
Repo-baseurl : http://192.168.50.160:81/whxlocalnetworkyum/
Repo-expire  : 21600 second(s) (last: Tue Apr 25 01:19:23 2023)
  Filter     : read-only:present
Repo-filename: /etc/yum.repos.d/whxlocalnetworkyum.repo

repolist: 2
[root@harbor1 yum.repos.d]#  yum repoinfo whxlocalnetworkyum  | grep pkgs
Failed to set locale, defaulting to C
Repo-pkgs    : 2
[root@harbor1 yum.repos.d]# cd /whxlocalyum/
[root@harbor1 whxlocalyum]# ls -al
total 3064
drwxrwxrwx   3 root root     160 Apr 25 01:23 .
dr-xr-xr-x. 24 root root    4096 Apr 24 23:36 ..
-rwxrwxrwx   1 root root 2849180 Apr  5 13:37 httpd-2.4.6-98.el7.centos.7.x86_64.rpm
-rwxrwxrwx   1 root root   96652 Apr  5 13:37 httpd-tools-2.4.6-98.el7.centos.7.x86_64.rpm
drwxrwxrwx   2 root root    4096 Apr 25 00:07 repodata
-rw-r--r--   1 root root  176896 Jun 11  2021 vsftpd-3.0.2-29.el7_9.x86_64.rpm
[root@harbor1 whxlocalyum]# createrepo --update /whxlocalyum/
Spawning worker 0 with 1 pkgs
Spawning worker 1 with 0 pkgs
Spawning worker 2 with 0 pkgs
Spawning worker 3 with 0 pkgs
Workers Finished
Saving Primary metadata
Saving file lists metadata
Saving other metadata
Generating sqlite DBs
Sqlite DBs complete
[root@harbor1 whxlocalyum]#  yum repoinfo whxlocalnetworkyum  | grep pkgs
Failed to set locale, defaulting to C
Repo-pkgs    : 2
[root@harbor1 whxlocalyum]# yum  clean all
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, langpacks
Cleaning repos: whxlocalnetworkyum
Cleaning up list of fastest mirrors
Other repos take up 491 M of disk space (use --verbose for details)
[root@harbor1 whxlocalyum]#  yum repoinfo whxlocalnetworkyum  | grep pkgs
Failed to set locale, defaulting to C
Repo-pkgs    : 3
[root@harbor1 whxlocalyum]# 
```

## 3. 使用局域网内yum源
```shell
[root@harbor1 whxlocalyum]# cd /etc/yum.repos.d/
[root@harbor1 yum.repos.d]# ls -al
total 20
drwxr-xr-x.   3 root root   48 Apr 25 01:26 .
drwxr-xr-x. 143 root root 8192 Apr 24 22:21 ..
drwxr-xr-x    2 root root 4096 Apr 25 01:26 bak
-rw-r--r--    1 root root  118 Apr 25 01:17 whxlocalnetworkyum.repo
[root@harbor1 yum.repos.d]# cat whxlocalnetworkyum.repo 
[whxlocalnetworkyum]
name=whxlocalnetworkyum
baseurl=http://192.168.50.160:81/whxlocalnetworkyum
gpgcheck=0
enabled=1
[root@harbor1 yum.repos.d]# yum install vsftpd
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
Resolving Dependencies
--> Running transaction check
---> Package vsftpd.x86_64 0:3.0.2-29.el7_9 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

=============================================================================================================================================================================================================================================
 Package                                             Arch                                                Version                                                       Repository                                                       Size
=============================================================================================================================================================================================================================================
Installing:
 vsftpd                                              x86_64                                              3.0.2-29.el7_9                                                whxlocalnetworkyum                                              173 k

Transaction Summary
=============================================================================================================================================================================================================================================
Install  1 Package

Total download size: 173 k
Installed size: 353 k
Is this ok [y/d/N]: y
Downloading packages:
vsftpd-3.0.2-29.el7_9.x86_64.rpm                                                                                                                                                                                      | 173 kB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : vsftpd-3.0.2-29.el7_9.x86_64                                                                                                                                                                                              1/1 
  Verifying  : vsftpd-3.0.2-29.el7_9.x86_64                                                                                                                                                                                              1/1 

Installed:
  vsftpd.x86_64 0:3.0.2-29.el7_9                                                                                                                                                                                                             

Complete!
[root@harbor1 yum.repos.d]# 
```



## 4. 如果docker里面没有vi，用root用户进入docker安装
```shell

[root@harbor1 whxlocalyum]# docker exec -it --user root 9f7784560008 bash
root@9f7784560008:/usr/local/apache2# apt-get update
Get:1 http://deb.debian.org/debian bullseye InRelease [116 kB]
Get:2 http://deb.debian.org/debian-security bullseye-security InRelease [48.4 kB]
Get:3 http://deb.debian.org/debian bullseye-updates InRelease [44.1 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 Packages [8183 kB]
Get:5 http://deb.debian.org/debian-security bullseye-security/main amd64 Packages [237 kB]
Get:6 http://deb.debian.org/debian bullseye-updates/main amd64 Packages [14.6 kB]
Fetched 8643 kB in 3s (2823 kB/s)                     
Reading package lists... Done
root@9f7784560008:/usr/local/apache2# apt-get install vim
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libgpm2 vim-common vim-runtime xxd
Suggested packages:
  gpm ctags vim-doc vim-scripts
The following NEW packages will be installed:
  libgpm2 vim vim-common vim-runtime xxd
0 upgraded, 5 newly installed, 0 to remove and 1 not upgraded.
Need to get 8174 kB of archives.
After this operation, 36.9 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
Get:1 http://deb.debian.org/debian bullseye/main amd64 xxd amd64 2:8.2.2434-3+deb11u1 [192 kB]
Get:2 http://deb.debian.org/debian bullseye/main amd64 vim-common all 2:8.2.2434-3+deb11u1 [226 kB]
Get:3 http://deb.debian.org/debian bullseye/main amd64 libgpm2 amd64 1.20.7-8 [35.6 kB]
Get:4 http://deb.debian.org/debian bullseye/main amd64 vim-runtime all 2:8.2.2434-3+deb11u1 [6226 kB]
Get:5 http://deb.debian.org/debian bullseye/main amd64 vim amd64 2:8.2.2434-3+deb11u1 [1494 kB]
Fetched 8174 kB in 1s (5621 kB/s)
debconf: delaying package configuration, since apt-utils is not installed
Selecting previously unselected package xxd.
(Reading database ... 7143 files and directories currently installed.)
Preparing to unpack .../xxd_2%3a8.2.2434-3+deb11u1_amd64.deb ...
Unpacking xxd (2:8.2.2434-3+deb11u1) ...
Selecting previously unselected package vim-common.
Preparing to unpack .../vim-common_2%3a8.2.2434-3+deb11u1_all.deb ...
Unpacking vim-common (2:8.2.2434-3+deb11u1) ...
Selecting previously unselected package libgpm2:amd64.
Preparing to unpack .../libgpm2_1.20.7-8_amd64.deb ...
Unpacking libgpm2:amd64 (1.20.7-8) ...
Selecting previously unselected package vim-runtime.
Preparing to unpack .../vim-runtime_2%3a8.2.2434-3+deb11u1_all.deb ...
Adding 'diversion of /usr/share/vim/vim82/doc/help.txt to /usr/share/vim/vim82/doc/help.txt.vim-tiny by vim-runtime'
Adding 'diversion of /usr/share/vim/vim82/doc/tags to /usr/share/vim/vim82/doc/tags.vim-tiny by vim-runtime'
Unpacking vim-runtime (2:8.2.2434-3+deb11u1) ...
Selecting previously unselected package vim.
Preparing to unpack .../vim_2%3a8.2.2434-3+deb11u1_amd64.deb ...
Unpacking vim (2:8.2.2434-3+deb11u1) ...
Setting up libgpm2:amd64 (1.20.7-8) ...
Setting up xxd (2:8.2.2434-3+deb11u1) ...
Setting up vim-common (2:8.2.2434-3+deb11u1) ...
Setting up vim-runtime (2:8.2.2434-3+deb11u1) ...
Setting up vim (2:8.2.2434-3+deb11u1) ...
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vim (vim) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vimdiff (vimdiff) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rvim (rvim) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/rview (rview) in auto mode
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/vi (vi) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/vi.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/vi.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/vi.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/vi.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/vi.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/vi.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/vi.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/vi.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group vi) doesn't exist
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/view (view) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/view.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/view.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/view.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/view.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/view.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/view.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/view.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/view.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group view) doesn't exist
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/ex (ex) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/ex.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/ex.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/ex.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/ex.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/ex.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/ex.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/ex.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/ex.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group ex) doesn't exist
update-alternatives: using /usr/bin/vim.basic to provide /usr/bin/editor (editor) in auto mode
update-alternatives: warning: skip creation of /usr/share/man/da/man1/editor.1.gz because associated file /usr/share/man/da/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/de/man1/editor.1.gz because associated file /usr/share/man/de/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/fr/man1/editor.1.gz because associated file /usr/share/man/fr/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/it/man1/editor.1.gz because associated file /usr/share/man/it/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ja/man1/editor.1.gz because associated file /usr/share/man/ja/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/pl/man1/editor.1.gz because associated file /usr/share/man/pl/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/ru/man1/editor.1.gz because associated file /usr/share/man/ru/man1/vim.1.gz (of link group editor) doesn't exist
update-alternatives: warning: skip creation of /usr/share/man/man1/editor.1.gz because associated file /usr/share/man/man1/vim.1.gz (of link group editor) doesn't exist
Processing triggers for libc-bin (2.31-13+deb11u5) ...

```