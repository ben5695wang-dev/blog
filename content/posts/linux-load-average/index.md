---
title: "Linux中的load average"
description: 
date: 2023-02-16T19:31:55+08:00
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
# 标签（Tags） - 适合具体关键词、多个细标签
tags:
  - Linux
  - load average
build:
    list: always    # Change to "never" to hide the page from the list
---

# Load average
本文阐述了在Linux中load average的发展过程、相关源代码和其意义
## 1.结论
### 1.1一句话结论
Linux中的load average，可以粗略理解为1分钟、5分钟、15分钟内正在运行的进程数的平均值。
### 1.2一段话结论
Linux中的load average，是统计状态为R和D的进程数的和，在1分钟、5分钟、15分钟内的三个平均值。
但是并不是真正的平均值，而是大约每5秒取一次值，使用指数移动平均算法，随着时间呈现的指数衰减的一个数值。
load average体现的是一个随着时间变化的系统负载的趋势，这个指标其实没什么用！
### 1.3如何判断这个值好坏
1.先看CPU数量
    CPU数量=物理CPU个数 * 每个物理CPU的核心数 * 2(如果开启了超线程)
```shell
# cat /proc/cpuinfo | grep "processor" | wc -l
```
2.查询处于(R)可执行状态和(D)不可中断睡眠状态的进程数
```shell
# ps -eo stat | grep ^[RD] | wc -l
```
3.然后用这个值除以CPU数量，得出一个值：得出的值不大于3的话那么系统的性能就是良好的，如果(每个CPU的任务数)大于5，那么这个系统可能会存在性能问题。
但是也仅仅是可能存在性能问题，因为下文可以看到，这个值可以很大，我在4c的虚拟机上将这个值跑到480也没问题。
原因是Linux的load average统计了处理R和D状态的运行数，因此这个值就不是一个只和CPU相关的值，还和I/O有关，所以当这个值偏大时，要看是什么引起了整个system层面的负载，而不一定是CPU。
## 2.进程
### 2.1进程的状态
在ps的帮助手册中可以看到进程状态的解释如下
```
Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to describe the state of a process:

        D    uninterruptible sleep (usually IO)
        R    running or runnable (on run queue)
        S    interruptible sleep (waiting for an event to complete)
        T    stopped by job control signal
        t    stopped by debugger during the tracing
        W    paging (not valid since the 2.6.xx kernel)
        X    dead (should never be seen)
        Z    defunct ("zombie") process, terminated but not reaped by its parent

For BSD formats and when the stat keyword is used, additional characters may be displayed:

        <    high-priority (not nice to other users)
        N    low-priority (nice to other users)
        L    has pages locked into memory (for real-time and custom IO)
        s    is a session leader
        l    is multi-threaded (using CLONE_THREAD, like NPTL pthreads do)
        +    is in the foreground process group
```
### 2.2查看进程的命令
```shell
# ps -eo stat | grep ^[RD] | wc -l
1
# ps -eo pid,time,cmd,stat|more
   PID     TIME CMD                         STAT
     1 00:00:12 /usr/lib/systemd/systemd -- Ss
     2 00:00:00 [kthreadd]                  S
     4 00:00:00 [kworker/0:0H]              S<
# ps -eLF | wc -l
455
#  ps axms | wc -l
649
# pstree
systemd-+-ModemManager---2*[{ModemManager}]
        |-NetworkManager---2*[{NetworkManager}]
......
        |-containerd-shim-+-start.sh-+-crond
        |                 |          `-sudo---rsyslogd---6*[{rsyslogd}]
        |                 `-10*[{containerd-shim}]
        |-containerd-shim-+-redis-server---4*[{redis-server}]
        |                 `-11*[{containerd-shim}]
......
# ps -ef --forest 
root       2996      1  0 02:36 ?        00:01:38 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root      12768   2996  0 07:38 ?        00:00:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 1514 -container-ip 172.18.0.2 -container-port 10514
root      13384   2996  0 07:38 ?        00:00:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.18.0.10 -container-port 8080
root      13391   2996  0 07:38 ?        00:00:00  \_ /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 80 -container-ip 172.18.0.10 -container-port 8080
gdm        4126      1  0 02:36 ?        00:00:00 dbus-launch --exit-with-session /usr/libexec/gnome-session-binary --autostart /usr/share/gdm/greeter/autostart
......
```
### 2.3cat /proc/loadavg
这个文件有五个值
```shell
# cat  /proc/loadavg
0.03 0.02 0.05 1/212 11932
```
第一个到第三个是1、5、15分钟的load average
第四个值是一个分数，分子是正在运行的进程数，分母是进程总数。分母可以通过以下命令得出：
```shell
# ps -eo stat | grep ^[RD] | wc -l
```
第五个值是最后进行的进程ID号
### 2.4查询进程数量相关命令
查询处于(R)可执行状态的进程数
```shell
# ps -eo stat|grep -e "R" | wc -l
```
查询处于(D)不可中断睡眠状态的进程数
```shell
# ps -eo stat|grep -e "D" | wc -l
```
查询处于(R)可执行状态和(D)不可中断睡眠状态的进程数
```shell
# ps -eo stat | grep ^[RD] | wc -l
# ps -eo stat|grep -e "R" -e "D"  | wc -l
```
## 3.CPU相关内容
### 3.1/proc/cpuinfo解释
processor : 逻辑CPU的编号，从0开始排序
vendor_id : CPU制造商
cpu family : CPU产品系列代号
physical id: 单个物理CPU编号，每个id值代表一个唯一的物理CPU
cpu cores : 单个物理CPU中封装的内核数
siblings : CPU核能处理的线程数。如果开启了超线程，则 siblings=2*cpu cores；没开启中则 siblings=cpu cores。CPU支持超线程技术，这意味着每个CPU核心可以同时处理多个线程。例如，如果有一个双核心处理器，并且每个核心都支持超线程，则siblings将显示4，即每个核心可以同时处理两个线程。
core id : 单个内核在其所处CPU中的编号，这个编号不一定连续
cpu MHz : CPU的实际使用主频
flags : 当前cpu支持的功能
```shell
# cat /proc/cpuinfo
processor       : 0
vendor_id       : GenuineIntel
cpu family      : 6
......
cpu MHz         : 2499.998
cache size      : 36608 KB
physical id     : 0
siblings        : 2
core id         : 0
cpu cores       : 1
```
计算方法:
物理CPU：physical id：物理cpu数量，不重复的 physical id 有几个就有多少个物理CPU。
核心数量：cpu cores：单个CPU里核心的数量。

逻辑CPU数=物理CPU个数×每颗核数     　　 #没开启超线程技术
逻辑CPU数=物理CPU个数×每颗核数 * 2  　  #开启了超线程技术
### 3.2查询CPU相关命令
一般情况下，使用processor的数量就可以，如果开了启线程，可以考虑再乘以2
CPU数量=物理CPU个数 * 每个物理CPU的核心数 * 2(如果开启了超线程)
查询物理CPU个数：
```shell
# cat  /proc/cpuinfo | grep "physical id" | sort | uniq
```
查询物理CPU核心数：
```shell
# cat /proc/cpuinfo | grep "cpu cores" | uniq
```
查询CPU总数量：
```shell
# cat /proc/cpuinfo | grep "processor" | wc -l
```
查询是否启用超线程技术：     //两者值相等则没开启超线程。
```shell
# cat /proc/cpuinfo | grep -e "cpu cores"  -e "siblings" | sort | uniq
```

## 4.load average的演变
### 4.1最开始的Load Averages--1973年08月10日
在1973年08月10日的TENEX Load Averages for July 1973中 https://datatracker.ietf.org/doc/html/rfc546
Load Averages这个值是处于R的状态的进程数在1、5、15分钟内的平均值。
这是使用LISP语言编写的，所以就不贴代码了。
The TENEX load average is a measure of CPU demand.  The load average is an average of the number of runable processes over a given time period.  For example, an ourly load average of 10 would mean that (for a single CPU system) at any time during that hour one could expect to see 1 process running and 9 others ready to run (i.e., not blocked for I/O) waiting for the CPU.

### 4.2Linux在0.96.a版本第一次引入这个功能--1992年05月22日
Linux在0.96.a版本中，修改了/kernel/sched.c文件，第一次引入这个功能，当时使用的是update_avg函数。
可以看出，当时统计处于R和D状态的进程数，使用的是指数移动平均算法。这和下文今天的Load Averages实现所使用的也是没什么太大区别。
/kernel/sched.c
344行：
[^_^]: 这里的averunnable[3]只定义了但没有显式赋值，原因是这个数组定义在文件中、函数外，因此会默认被赋值0。https://stackoverflow.com/questions/76082953/in-c-language-if-the-array-members-is-not-assigned-a-value-when-they-are-initia
```c
#define	FSHIFT	11
#define	FSCALE	(1<<FSHIFT)
/*
 * Constants for averages over 1, 5, and 15 minutes
 * when sampling at 5 second intervals.
 */
static unsigned long cexp[3] = {
	1884,	/* 0.9200444146293232 * FSCALE,	 exp(-1/12) */
	2014,	/* 0.9834714538216174 * FSCALE,	 exp(-1/60) */
	2037,	/* 0.9944598480048967 * FSCALE,	 exp(-1/180) */
};
unsigned long averunnable[3];	/* fixed point numbers */

void update_avg(void)
{
    	int i, n=0;
	struct task_struct **p;

	for(p = &LAST_TASK; p > &FIRST_TASK; --p)
		if (*p && ((*p)->state == TASK_RUNNING || 
			   (*p)->state == TASK_UNINTERRUPTIBLE))
			++n;
	
	for (i = 0; i < 3; ++i)
		averunnable[i] = (cexp[i] * averunnable[i] +
			n * FSCALE * (FSCALE - cexp[i])) >> FSHIFT;
}
```
### 4.3Linux在0.99.13版本修改为只统计处于R状态的进程数--1993年09月20日
在0.99.13版本中，因为GNU的关系，修改了这个函数及其实现。
/kernel/sched.c
362行：
[^_^]:在这里，声明avenrun[3]的时候同时做了初始化。
```c
/*
 * Hmm.. Changed this, as the GNU make sources (load.c) seems to
 * imply that avenrun[] is the standard name for this kind of thing.
 * Nothing else seems to be standardized: the fractional size etc
 * all seem to differ on different machines.
 */
unsigned long avenrun[3] = { 0,0,0 };

/*
 * Nr of active tasks - counted in fixed-point numbers
 */
static unsigned long count_active_tasks(void)
{
	struct task_struct **p;
	unsigned long nr = 0;

	for(p = &LAST_TASK; p > &FIRST_TASK; --p)
		if (*p && (*p)->state == TASK_RUNNING)
			nr += FIXED_1;
	return nr;
}

static inline void calc_load(void)
{
	unsigned long active_tasks; /* fixed-point */
	static int count = LOAD_FREQ;

	if (count-- > 0)
		return;
	count = LOAD_FREQ;
	active_tasks = count_active_tasks();
	CALC_LOAD(avenrun[0], EXP_1, active_tasks);
	CALC_LOAD(avenrun[1], EXP_5, active_tasks);
	CALC_LOAD(avenrun[2], EXP_15, active_tasks);
}
```
/kernel/sched.h里面的相关代码片段
```c
#define CALC_LOAD(load,exp,n) \
	load *= exp; \
	load += n*(FIXED_1-exp); \
	load >>= FSHIFT;
```
### 4.4Linux在0.99.14版本将状态处于D的进程数也纳入统计--1993年10月29日
在0.99.14版本中，在1993年10月29日，Matthias认为只统计处于R状态的进程数量，没有真实反应系统的负载，所以应该将处于D状态的进程也纳入统计。
例如将系统的一块SSD磁盘换成一块7.2k转的机械盘，那么系统相对之前会变的很慢，这是因为很多进程会处于D状态，等待I/O(磁盘I/O、FileSystems I/O、网络I/O[网络文件系统]等)，那么只统计处于R的进程数，就没有真实反应系统的负载--因为系统的确变慢了！所以应该将处于D状态的进程也纳入统计，并提交了代码。
/kernel/sched.c
418行：
```c
static unsigned long count_active_tasks(void)
{
        struct task_struct **p;
        unsigned long nr = 0;

        for(p = &LAST_TASK; p > &FIRST_TASK; --p)
                if (*p && ((*p)->state == TASK_RUNNING ||
                           (*p)->state == TASK_UNINTERRUPTIBLE ||
                           (*p)->state == TASK_SWAPPING))
                        nr += FIXED_1;
        return nr;
}

static inline void calc_load(void)
{
        unsigned long active_tasks; /* fixed-point */
        static int count = LOAD_FREQ;

        if (count-- > 0)
                return;
        count = LOAD_FREQ;
        active_tasks = count_active_tasks();
        CALC_LOAD(avenrun[0], EXP_1, active_tasks);
        CALC_LOAD(avenrun[1], EXP_5, active_tasks);
        CALC_LOAD(avenrun[2], EXP_15, active_tasks);
}
```
也就是将原来只统计处于RUNNING状态的进程数，变为统计处于RUNNING、UNINTERRUPTIBLE、TASK_SWAPPING这三种状态的进程数量

From: Matthias Urlichs <urlichs@smurf.sub.org>
Subject: Load average broken ?
Date: Fri, 29 Oct 1993 11:37:23 +0200

The kernel only counts "runnable" processes when computing the load average. I don't like that; the problem is that processes which are swapping or waiting on "fast", i.e. noninterruptible, I/O, also consume resources.
It seems somewhat nonintuitive that the load average goes down when you replace your fast swap disk with a slow swap disk...
Anyway, the following patch seems to make the load average much more consistent WRT the subjective speed of the system. And, most important, the load is still zero when nobody is doing anything. ;-)


### 4.5最新版Linux中的Load Averages：一个很SB的指标，但是听说它很重要--2022年02月22日
注意看loadavg.c文件中的第一段注释：这是计算全局负载的指标，它其实很SB，但是不知道为什么他们都认为这玩意很重要。
在算法实现上，因为现在的机器可能拥有大量的CPU，那么计算起来又费时又费力，从而改为了分布式和异步方式。
同时也说

 ```c
// SPDX-License-Identifier: GPL-2.0
/*
 * kernel/sched/loadavg.c
 *
 * This file contains the magic bits required to compute the global loadavg
 * figure. Its a silly number but people think its important. We go through
 * great pains to make it work on big machines and tickless kernels.
 */

/*
 * Global load-average calculations
 *
 * We take a distributed and async approach to calculating the global load-avg
 * in order to minimize overhead.
 *
 * The global load average is an exponentially decaying average of nr_running +
 * nr_uninterruptible.
 *
 * Once every LOAD_FREQ:
 *
 *   nr_active = 0;
 *   for_each_possible_cpu(cpu)
 *	nr_active += cpu_of(cpu)->nr_running + cpu_of(cpu)->nr_uninterruptible;
 *
 *   avenrun[n] = avenrun[0] * exp_n + nr_active * (1 - exp_n)
 *
 * Due to a number of reasons the above turns in the mess below:
 *
 *  - for_each_possible_cpu() is prohibitively expensive on machines with
 *    serious number of CPUs, therefore we need to take a distributed approach
 *    to calculating nr_active.
 *
 *        \Sum_i x_i(t) = \Sum_i x_i(t) - x_i(t_0) | x_i(t_0) := 0
 *                      = \Sum_i { \Sum_j=1 x_i(t_j) - x_i(t_j-1) }
 *
 */
 ```

## 5.测试
写一段程序，生成N个一直占用CPU资源的进程，运行T秒，然后观察load-average的值。
可以看到运行1分钟以上后，load-average的第一个值和进程数接近。
```shell
# gcc -o forkpid forkpid.c
# ./forkpid 500 180
```

在这里运行了500个进程，运行了180秒，可以看到top中的load-average的第一值为482。

```shell
# top
top - 09:32:50 up 13:16,  5 users,  load average: 482.00, 283.16, 128.12
Tasks: 771 total, 501 running, 270 sleeping,   0 stopped,   0 zombie
%Cpu(s): 99.5 us,  0.2 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  0.3 si,  0.0 st
KiB Mem :  7989848 total,  5106864 free,  1373304 used,  1509680 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  6162116 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND    
 26609 root      20   0    4216     88      0 R   1.5  0.0   0:01.55 forkpid    
 26503 root      20   0    4216     88      0 R   1.2  0.0   0:01.56 forkpid    
 26505 root      20   0    4216     88      0 R   1.2  0.0   0:01.55 forkpid    
 26506 root      20   0    4216     88      0 R   1.2  0.0   0:01.55 forkpid    
 26507 root      20   0    4216     88      0 R   1.2  0.0   0:01.55 forkpid    
 26509 root      20   0    4216     88      0 R   1.2  0.0   0:01.55 forkpid    
```

### 5.1测试代码
```c
# cat forkpid.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>
#include <signal.h>

//define a signal handler function to terminate child processes
void handler(int sig)
{
    printf("Child process %d received signal %d\n", getpid(), sig);
    exit(0);
}

int main(int argc, char *argv[])
{
    if(argc != 3) //check the number of arguments
    {
        printf("Usage: %s <number of processes> <time to run>\n", argv[0]); //print the usage
        exit(1);
    }

    int N = atoi(argv[1]); //convert the first argument to an integer, indicating the number of processes to create
    int T = atoi(argv[2]); //convert the second argument to an integer, indicating the time to run

    int i;
    for(i = 0; i < N; i++) //loop N times
    {
        pid_t pid; //define a process ID variable
        pid = fork(); //create a child process

        if(pid == -1) //if creation fails, return -1
        {
            perror("fork error");
            exit(1);
        }

        else if(pid == 0) //if creation succeeds, return 0, indicating child process
        {
            printf("This is child process %d, pid is %d\n", i+1, getpid()); //print the child process number and ID
            signal(SIGALRM, handler); //register the signal handler function to receive the SIGALRM signal sent by the parent process
            for(;;) //execute an infinite loop to consume CPU resources
            {
                double x = 3.14159; //define a double variable
                x = x * x; //calculate the square of x
            }
        }
    }

    //parent process waits for T seconds and then sends SIGALRM signal to all child processes to terminate them
    sleep(T); //wait for T seconds
    kill(0, SIGALRM); //send SIGALRM signal to all processes in the same process group
    printf("Parent process sent signal %d to all child processes\n", SIGALRM);
    //parent process waits for all child processes to end and gets their return values
    
    int status; //define a status variable
    while(wait(&status) != -1) //if there are still child processes not finished, continue to wait
    {
        printf("Child process exited with status %d\n", status); //print the exit status of the child process
    }
    return 0;
}

```



## 6.相关代码和测试全过程
[Linux中的load average相关代码和测试过程](http://39.105.160.122/2023/04/24/load-average%e6%9f%a5%e5%85%b3%e4%bb%a3%e7%a0%81%e5%92%8c%e6%b5%8b%e8%af%95%e8%bf%87%e7%a8%8b/ "Linux中的load average相关代码和测试过程")
