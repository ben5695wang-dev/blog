---
title: "Linux诊断和故障排除系列(七) -- 应用程序诊断"
description: 
date: 2024-07-07T20:02:29+08:00
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
  - 应用程序诊断
build:
    list: always    # Change to "never" to hide the page from the list
---

# 1. 确定第三方软件的库依赖关系 
## 1.1 什么是库依赖
第三方软件和第一方软件等，会共享很多function，而这些通常是通过库来实现的，所以将这些通用代码(例如读、写、打开文件等)放到一个共享库中，然后每个APP访问这个共享库。 例如第三方APP是nginx，第一方APP是pmlogger，这两个APP会共享一些不同的库，例如libcrypto。
为了让nginx和pmlogger访问libcrypto，使用的是ld-linker工具，它负责加载共享库，并将共享库映射到内存中，然后APP就可以访问这些共享库了。
## 1.2 查询使用了哪些库
ldconfig -p //查看在内存cache中，哪些库是可用的
ldd //显示共享库依赖，需要提供 可执行文件的全路径
ldd $(which python3) //将which python3命令的输出，作为ldd命令的参数
```shell
$ ldconfig -p | more
298 libs found in cache `/etc/ld.so.cache`
        p11-kit-trust.so (libc6,x86-64) => /lib64/p11-kit-trust.so
        libz.so.1 (libc6,x86-64) => /lib64/libz.so.1
        libyaml-0.so.2 (libc6,x86-64) => /lib64/libyaml-0.so.2
......
$ which python3
/bin/python3
$ ldd /bin/python3
        linux-vdso.so.1 =>  (0x00007ffd177ad000)
        libpython3.6m.so.1.0 => /lib64/libpython3.6m.so.1.0 (0x00007fcb5945e000)
......
$ ldd $(which python3)              //功能等同于上面二条命令
```
## 1.3 共享库问题
共享库问题，如果共享库有问题，可能就会输出共享库名，但库的位置可能无法提供(例如闪烁？)
dnf whatprovides liblzma //哪个软件包安装了这个库，然后 安装软件包就OK了
```shell
$ ldd /bin/python3
        linux-vdso.so.1 =>  (0x00007ffd177ad000)
        libpython3.6m.so.1.0 => /lib64/libpython3.6m.so.1.0 (0x00007fcb5945e000)
$ dnf whatprovides liblzma		//这样可能什么也不返回
$ dnf whatprovides *liblzma*		//加通配符，这样可能返回太多，包括文档之类的
$ yum whatprovides *libc.so*|wc 
    595    1598   19436
$ dnf whatprovides */liblzma.so*		//库都是 .so的，
$ yum whatprovides */libc.so*|wc
    222     508    5008
$ dnf reinstall xz-devel                //用dnf/yum重新安装这个包就可以了
```
# 2. 确定应用程序是否存在内存泄漏
并不是解决内存泄露，而是发现内存泄露并生成报告，从而让开发去处理。
valgrind: Linux可执行文件的debugging和profiling套件，主要用于检测 C 和 C++ 程序中的内存问题。
--tool=memcheck //内存工具(也是默认的，不定义也是这个)
--log-file=	//输出到自定义文件
--leak-check=full //检查所有内存泄露，而不是占用额外的内存，这将提供完整内存泄露细节
(对于其他语言，Java内存泄露检测工具：VisualVM：(jvisualvm.exe)。Python内存泄露检测工具：memory_profiler。Go语言内存泄露检测工具：go-leak。)
```shell
dnf install valgrind
curl https://github.com/linuxacademy/content-rh342/raw/main/lessons/section-7/memleak_test_app -O
chmod +x memleak_test_app
valgrind ./memleak_test_app //需要全路径或相对路径名
valgrind --leak-check=full /root/memleak_test_app //输出所有细节
valgrind --leak-check=full --show-leak-kinds=all /root/memleak_test_app //输出 内存泄露 更详细的细节。会报告有多少内存被使用，但没有被reallocated
valgrind --leak-check=full --show-leak-kinds=all --log-file=./memcheck /root/memleak_test_app //输出到文件
```
```shell
$ valgrind /root/memleak_test_app               
==10077== Memcheck, a memory error detector
==10077== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==10077== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==10077== Command: /root/memleak_test_app
==10077== 
==10077== Invalid write of size 4
==10077==    at 0x40053B: f (in /root/memleak_test_app)
==10077==    by 0x40054B: main (in /root/memleak_test_app)
==10077==  Address 0x5205068 is 0 bytes after a block of size 40 alloc'd
==10077==    at 0x4C29F73: malloc (vg_replace_malloc.c:309)
==10077==    by 0x40052E: f (in /root/memleak_test_app)
==10077==    by 0x40054B: main (in /root/memleak_test_app)
==10077== 
==10077== 
==10077== HEAP SUMMARY:
==10077==     in use at exit: 40 bytes in 1 blocks
==10077==   total heap usage: 1 allocs, 0 frees, 40 bytes allocated
==10077== 
==10077== LEAK SUMMARY:
==10077==    definitely lost: 40 bytes in 1 blocks
==10077==    indirectly lost: 0 bytes in 0 blocks
==10077==      possibly lost: 0 bytes in 0 blocks
==10077==    still reachable: 0 bytes in 0 blocks
==10077==         suppressed: 0 bytes in 0 blocks
==10077== Rerun with --leak-check=full to see details of leaked memory      <<<<<<
==10077== 
==10077== For lists of detected and suppressed errors, rerun with: -s
==10077== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```
# 3. 使用标准工具调试应用程序 
和上一节一样，这一节的目的是找出问题，而不是解决问题。很少有能解决问题的情况，只有例如修改文件权限等。
ltrace和strace它们有相同的参数
## 3.1  ltrace
追踪命令或可执行文件的库调用。-o输出到自定义文件，-p自定义进程ID，-e通过表达式过滤从而只关注特定的库
```shell
$ dnf install lstrace
$ touch empty
$ cat empty
$ ltrace $(which cat) empty                                                 
__libc_start_main(0x401a20, 2, 0x7ffd7267e818, 0x408ca0 <unfinished ...>   
getpagesize()                                                              = 4096
strrchr("/bin/cat", '/')                                                   = "/cat"
setlocale(LC_ALL, "")                                                      = "en_US.UTF-8"
bindtextdomain("coreutils", "/usr/share/locale")                           = "/usr/share/locale"
textdomain("coreutils")                                                    = "coreutils"
__cxa_atexit(0x4029f0, 0, 0, 0x736c6974756572)                             = 0
getopt_long(2, 0x7ffd7267e818, "benstuvAET", 0x60bc60, nil)                = -1
__fxstat(1, 1, 0x7ffd7267e660)                                             = 0
open("empty", 0, 037777600000)                                             = 3                      <<<打开文件
__fxstat(1, 3, 0x7ffd7267e660)                                             = 0
posix_fadvise(3, 0, 0, 2)                                                  = 0
malloc(69631)                                                              = 0x76e030               <<<分配内存
read(3, "", 65536)                                                         = 0                      <<<读取文件
free(0x76e030)                                                             = <void>
close(3)                                                                   = 0                      <<<关闭文件
exit(0 <unfinished ...>                                                    
__fpending(0x7ff0582ee400, 0, 64, 0x7ff0582eeeb0)                          = 0
......
+++ exited (status 0) +++
$ echo "hello" > empty
$ ltrace $(which cat) empty
__libc_start_main(0x401a20, 2, 0x7ffe8d6626f8, 0x408ca0 <unfinished ...>
getpagesize()                                                              = 4096
strrchr("/bin/cat", '/')                                                   = "/cat"
setlocale(LC_ALL, "")                                                      = "en_US.UTF-8"
bindtextdomain("coreutils", "/usr/share/locale")                           = "/usr/share/locale"
textdomain("coreutils")                                                    = "coreutils"
__cxa_atexit(0x4029f0, 0, 0, 0x736c6974756572)                             = 0
getopt_long(2, 0x7ffe8d6626f8, "benstuvAET", 0x60bc60, nil)                = -1
__fxstat(1, 1, 0x7ffe8d662540)                                             = 0
open("empty", 0, 037777600000)                                             = 3
__fxstat(1, 3, 0x7ffe8d662540)                                             = 0
posix_fadvise(3, 0, 0, 2)                                                  = 0
malloc(69631)                                                              = 0x260a030
read(3, "hello\n", 65536)                                                  = 6                      <<<读取文件内容
write(1, "hello\n", 6hello)                                                = 6                      <<<写文件内容到标准输出
read(3, "", 65536)                                                         = 0                      <<<读取文件内容
free(0x260a030)                                                            = <void>
close(3)                                                                   = 0
exit(0 <unfinished ...>
__fpending(0x7fc1ce1aa400, 0, 64, 0x7fc1ce1aaeb0)                          = 0
fileno(0x7fc1ce1aa400)                                                     = 1
......
+++ exited (status 0) +++
$ ltrace -e read $(which cat) empty                                                     //过滤指定库函数
cat->read(3, "hello\n", 65536)                                             = 6
hello
cat->read(3, "", 65536)                                                    = 0
+++ exited (status 0) +++
```
## 3.2 strace

追踪命令或可执行文件的系统调用。-o输出到自定义文件，-p自定义进程ID，-e通过表达式过滤从而只关注特定的系统调用
```shell
$ dnf install strace -y
$ strace $(which cat) empty  
execve("/bin/cat", ["/bin/cat", "empty"], 0x7ffdf6856388 /* 24 vars */) = 0
brk(NULL)                               = 0x13de000
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8972c84000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=22608, ...}) = 0
mmap(NULL, 22608, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f8972c7e000
close(3)                                = 0
open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0`&\2\0\0\0\0\0"..., 832) = 832
fstat(3, {st_mode=S_IFREG|0755, st_size=2156592, ...}) = 0
mmap(NULL, 3985920, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f8972696000
mprotect(0x7f897285a000, 2093056, PROT_NONE) = 0
mmap(0x7f8972a59000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1c3000) = 0x7f8972a59000
mmap(0x7f8972a5f000, 16896, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f8972a5f000
close(3)                                = 0
mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8972c7d000
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8972c7b000
arch_prctl(ARCH_SET_FS, 0x7f8972c7b740) = 0
access("/etc/sysconfig/strcasecmp-nonascii", F_OK) = -1 ENOENT (No such file or directory)
access("/etc/sysconfig/strcasecmp-nonascii", F_OK) = -1 ENOENT (No such file or directory)
mprotect(0x7f8972a59000, 16384, PROT_READ) = 0
mprotect(0x60b000, 4096, PROT_READ)     = 0
mprotect(0x7f8972c85000, 4096, PROT_READ) = 0
munmap(0x7f8972c7e000, 22608)           = 0
brk(NULL)                               = 0x13de000
brk(0x13ff000)                          = 0x13ff000
brk(NULL)                               = 0x13ff000
open("/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=106176928, ...}) = 0
mmap(NULL, 106176928, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f896c153000
close(3)                                = 0
fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(136, 0), ...}) = 0
open("empty", O_RDONLY)                 = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=6, ...}) = 0
fadvise64(3, 0, 0, POSIX_FADV_SEQUENTIAL) = 0
read(3, "hello\n", 65536)               = 6
write(1, "hello\n", 6hello
)                  = 6
read(3, "", 65536)                      = 0
close(3)                                = 0
close(1)                                = 0
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++
$ strace -e read $(which cat) empty 
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0`&\2\0\0\0\0\0"..., 832) = 832
read(3, "hello\n", 65536)               = 6
hello
read(3, "", 65536)                      = 0
+++ exited with 0 +++
```
# 4. SELinux
## 4.1 SELinux上下文
SELinux Context是围绕特定文件的安全架构和对于特定文件的访问控制
setools: 包括SELinux相关的troubleshooting工具的包
查看特定文件的相关上下文，需要使用这个setools
ls -Z /var/log/pcp/ //在安装setools以后，就可以使用-z的ls命令来查看文件的SELinux上下文信息
semange
semange login -l		//显示用户，列出用户映射
semange user -l 		//显示与用户相关的角色
semange fcontext -l	//显示上下文类型关联
semange port -l			//端口映射
seinfo
seinfo -u		//显示用户，列出用户映射
seinfo -r 		//显示与用户相关的角色
seinfo -t		//显示上下文类型关联
seinfo -p		//端口映射
```shell
$ dnf install setool -y
$ ls -Z /var/log/pcp/                       //在安装setools以后，就可以使用-z的ls命令来查看文件的SELinux上下文信息
system_u:object_r:pcp_log_t:s0 pmcd         //是一个system_user对象，object_role，是pcp_log_type，s0是多层、多类安装设置
$ ls -Z /etc/httpd/
$ seinfo -u 	//显示用户
$ semange login -l
$ seinfor -r		//显示角色
$ semange user -l
$ semange port -l
```
## 4.2 识别与SELinux相关的问题 
### 4.2.1 测试问题是否出在SELinux上
对于SELinux的PD的第一步，是测试问题是否出在SELinux上。
运行 setenforce 0，然后测试文件是否还存在，如果还存在，那就和无关；如果没有问题了，那就是SELinux的问题。
### 4.2.1 SElinux相关日志
相关日志：
```shell
tail /var/log/audit/audit.log
sealert -a /var/log/audit/audit.log //过滤只有SE相关的错误
ausearch -m avc -ts recent //搜索avc或最宾发生的访问向量缓存问题
restorecon //恢复SELinux上下文，-R递归，-p/v输出(print/verbose)
Booleans //修改SELinux组件的开关
getsebool //显示boolean信息，-a显示所有
setsebool //设置Boolean，-P永久设置(默认是临时的)
```
## 4.3 解决与SELinux相关的问题 
### 4.3.1 制造问题
```shell
$ ls -Z /etc/httpd/
$ ls -Z /var/log/httpd/
$ chcon -R --reference /etc/httpd/ /var/log/httpd			//将/var/log/httpd的上下文修改为/etc/httpd/的样子
$ vi /etc/httpd/conf/httpd.conf
DocumentRoot "/home/cloud_user/www"			//修改这一行
$ mkdir /home/cloud_user/www
$ echo "hihihello"> /home/cloud_user/www/index.html
$ chown -R cloud_user:cloud_user /home/cloud_user/www/
$ chmod -R 755 /home/cloud_user/www/
$ rm -rf /etc/httpd/conf.d/welcome.conf
$ systemcl restart httpd		//会收到错误
```
### 4.3.2 解决问题
```shell
$ tail /var/log/audit/audit.log				//查看是否有avc相关
$ sealert -a /var/log/audit/audit.log			//过滤只有SE相关的错误
$ ausearch -m avc -ts recent				//搜索avc或最近发生的访问向量缓存问题
$ find / -inum 4504376						//查找inode为4504376的文件
$ restorecon -Rv /var/log/httpd/            //递归恢复SELinux上下文。这里不使用chcon是因为不知道应该修改为什么
$ systemctl start httpd                     //一切正常
$ curl localhost			//会得到403 Forbidden的错误
$ ausearch -m avc -ts recent				//搜索avc或最近发生的访问向量缓存问题，找httpd相关的
$ sealert -a /var/log/audit/audit.log			//这里可能会给出进一步确定问题的提示，可能有setsebool相关命令的建议
$ getsebool -a
$ setsebool httpd_enable_homedirs 1		//打开，on
$ getsebool htpdd_enable_homedirs		//查看
$ curl localhost										//一切正常
$ setsebool -P httpd_enable_homedirs 1		//永久
```