---
title: "阿里云的vCPU和CPU有什么区别？-阿里云开发者社区"
tags:
  - "clippings literature-note"
date: 2025-02-27
time: 2025-02-27T13:17:51+08:00
source: "https://developer.aliyun.com/article/1205518"
---
**简介：** 阿里云的vCPU和CPU有什么区别？阿里云服务器vCPU和CPU是什么意思？CPU和vCPU有什么区别？一台云服务器ECS实例的CPU选项由CPU物理核心数和每核线程数决定，CPU是中央处理器，一个CPU可以包含若干个物理核，通过超线程HT（Hyper-Threading）技术可以将一个物理核变成两个逻辑处理核。vCPU（virtual CPU）是ECS实例的虚拟处理核。云服务器吧来详细说下阿里云服务器CPU和vCPU的区别：

阿里云服务器vCPU和CPU是什么意思？CPU和vCPU有什么区别？一台云服务器ECS实例的CPU选项由CPU物理核心数和每核线程数决定，CPU是中央处理器，一个CPU可以包含若干个物理核，通过超线程HT（Hyper-Threading）技术可以将一个物理核变成两个逻辑处理核。vCPU（virtual CPU）是ECS实例的虚拟处理核。云服务器吧来详细说下阿里云服务器CPU和vCPU的区别：

## 0.1 云服务器的CPU和vCPU有什么区别？

CPU是指中央处理器，CPU代表物理CPU核数，是真实存在的，CPU不是虚拟的，vCPU是物理CPU的基础上通过超线程HT技术虚拟出来的，一般来讲，云服务器的CPU指的是vCPU。

首先一台[云服务器ECS](https://www.aliyun.com/product/ecs?source=5176.11533457&userCode=r3yteowb)实例的CPU选项由CPU物理核心数和每核线程数决定，以ecs.g6.xlarge实例为例，阿里云默认提供2个物理核CPU，开启超线程，将每核线程数设置为2，则该实例规格有2\*2=4个vCPU，默认情况下该实例规格开启超线程配置。如果关闭超线程配置，则1个物理核只能运行1个线程，实例的vCPU数量等于物理核数，为2。

## 0.2 CPU与vCPU

关于[阿里云服务器ECS](https://www.aliyun.com/product/ecs?source=5176.11533457&userCode=r3yteowb)的CPU和vCPU的官方解释：CPU是中央处理器，一个CPU可以包含若干个物理核，通过超线程HT（Hyper-Threading）技术可以将一个物理核变成两个逻辑处理核。vCPU（virtual CPU）是ECS实例的虚拟处理核。

阿里云ECS的超线程的实现基于x86平台架构HT技术，允许在一个物理核上并发地运行两个线程（Thread），一个线程可以视为一个vCPU。

下表从多个维度对比了ECS实例的CPU选项参数。  
![cpuvcpu.jpg](https://ucc.alicdn.com/pic/developer-ecology/gwu3lhygmapgi_e2c3cae332d641828f8768b17e1cc092.jpg?x-oss-process=image/resize,w_1400/format,webp "cpuvcpu.jpg")

## 0.3 支持自定义CPU选项的ECS实例规格

[阿里云服务器ECS](https://www.aliyun.com/product/ecs?source=5176.11533457&userCode=r3yteowb)有多种实例规格，以下ECS实例支持自定义CPU选项：

- 通用型：g7a、g7、g7t、g7ne、g6t、g6a、g6e、g6
- 计算型：c7a、c7、c7t、c6t、c6a、c6e、c6
- 内存型：r7a、r7、r7t、re6p、r6a、r6e、r6
- 高主频型：hfg7、hfc7、hfr7、hfg6、hfc6、hfr6
- 本地SSD型：i3g、i3
- 可以使用阿里云测速工具 aliyunping.com 测试一下本地到阿里云服务器各个地域节点的Ping值网络延迟
- 更多关于阿里云服务器ECS处理器说明，请以官方页面为准：[https://www.aliyun.com/product/ecs](https://www.aliyun.com/product/ecs?source=5176.11533457&userCode=r3yteowb)

可以在云服务器ECS实例规格表中查看各实例规格CPU物理核心数（CpuOptions.Core）与每核线程数（CpuOptions.ThreadsPerCore）的默认值和取值范围，未列出的实例规格不支持自定义CPU选项。

更多关于阿里云服务器ECS处理器说明，请以官方页面为准：[https://www.aliyun.com/product/ecs](https://www.aliyun.com/product/ecs?source=5176.11533457&userCode=r3yteowb)

目录

- [云服务器的CPU和vCPU有什么区别？](https://developer.aliyun.com/article/#slide-0)
- [CPU与vCPU](https://developer.aliyun.com/article/#slide-1)
- [支持自定义CPU选项的ECS实例规格](https://developer.aliyun.com/article/#slide-2)