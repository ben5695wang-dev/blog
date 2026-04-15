---
title: "Linux性能诊断和调优系列(九)--计算密集型应用性能调优案例"
description: 
date: 2024-06-20T19:31:55+08:00
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
  - Linux性能诊断和调优系列
# 标签（Tags） - 适合具体关键词、多个细标签
tags:
  - Linux
  - Linux性能诊断和调优系列
  - 计算密集型应用性能调优案例
build:
    list: always    # Change to "never" to hide the page from the list
---


Linux性能诊断和调优系列(九)--计算密集型应用性能调优案例
![](Linux_performance_09.png)
# 目录
计算密集型常见的场景
操作系统级CPU参数
使用slice配置应用程序CPU资源
将应用程序绑定在指定CPU上
为应用程序指定CPU调度策略和优先级
# 计算密集型常见的场景
数据仓库：进行ETL（提取、转换、加载）操作和数据聚合。
大规模事务处理：处理大量并发事务的系统，如在线交易平台。
实时分析：需要快速响应时间的实时数据处理和分析。
财务建模和风险分析：进行复杂的金融算法和风险评估。
科学计算和模拟，例如气候模型、物理模拟、分子动力学模拟等；
高性能计算（HPC）、人工智能和深度学习等。
所以对于计算密集型应用案例，我们重点关注CPU方面的调整。
# 操作系统级CPU参数
对于计算密集型应用案例，常见的CPU方面的调整参数如下：
energy_performance_preference，这个参数用于设置系统整体的能耗与性能，建议使用performance，也就是最佳性能优先。
energy_perf_bias，这个参数用于设置CPU的能耗与性能，建议使用performance，也就是最佳性能优先。
governor，这个参数用于设置CPU频率策略，建议使用performance，从而让CPU尽可能保持在最高频率。
min_perf_pct，这个参数用于设置CPU性能的最小百分比，100表示CPU始终以最大性能运行。
# 使用slice配置应用程序CPU资源
如果在一个操作系统中，有多个应用程序，为了保证某个应用程序或限制某个应用程序，可以使用slice来配置应用程序可以使用的CPU资源。
在slice文件中，常见设置如下：
CPUQuota，这个参数用于设置可以使用的CPU资源百分比，例如40%。
CPUAccounting，这个参数用于设置开启对CPU使用的监控。
# 将应用程序绑定在指定CPU上
CPUAffinity，这个参数用于设置CPU的关联性，可以将应用程序绑定在指定CPU上，例如CPUAffinity=0，从而增加应用程序的性能。
# 为应用程序指定CPU调度策略和优先级
CPUSchedulingPolicy，这个参数用于设置CPU调度策略，常用的有NORMAL，BATCH和IDLE。
CPUScchedulingPriority，这个参数用于设置进程优先级，范围从 -20到+19，值越小优先级越高。
# 更多内容请参见本系列其他文章
<<Linux性能诊断和调优系列(一)--30秒3条命令诊断Linux性能瓶颈>>
<<Linux性能诊断和调优系列(二)--CPU篇>>
<<Linux性能诊断和调优系列(三)--内存篇>>
<<Linux性能诊断和调优系列(四)--硬盘篇>>
<<Linux性能诊断和调优系列(五)--文件系统篇>>
<<Linux性能诊断和调优系列(六)--网络篇>>
<<Linux性能诊断和调优系列(七)--虚拟机及容器篇>>
<<Linux性能诊断和调优系列(八)--虚拟环境性能调优案例>>
<<Linux性能诊断和调优系列(九)--计算密集型应用性能调优案例>>
<<Linux性能诊断和调优系列(十)--存储密集型应用性能调优案例>>
<<Linux性能诊断和调优系列(十一)--大内存型应用性能调优案例>>

本文内容为原创，如需转载，请务必注明原文出处。
更多相关内容，欢迎访问我的个人网站：hongxu.wang。
我们还提供免费的技术支持，欢迎通过公众号与我们联系。