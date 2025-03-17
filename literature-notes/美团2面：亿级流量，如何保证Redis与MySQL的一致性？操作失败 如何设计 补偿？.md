---
title: "美团2面：亿级流量，如何保证Redis与MySQL的一致性？操作失败 如何设计 补偿？"
tags:
  - "clippings literature-note"
date: 2025-03-16
time: 2025-03-16T15:26:07+08:00
source: "https://mp.weixin.qq.com/s/_VyHzICG_qZENjjnHGC0UA"
is_archie: "false"
---
原创 45岁老架构师尼恩 *2025年03月16日 13:21*

## 说在前面

在40岁老架构师 尼恩的**读者交流群**(50+)中，最近有小伙伴拿到了一线互联网企业如阿里、滴滴、极兔、有赞、希音、百度、网易、美团的面试资格，遇到很多很重要的redis一致性面试题，类似如下：

> - 如何保障 MySQL 和 Redis 的数据一致性？
> - 如何保障 MySQL 和 Cache 的数据一致性？
> - 双十一大促中，如何保证Redis与MySQL的最终一致性？
> - 若因网络抖动导致缓存更新失败，如何设计补偿机制？
> - 在双十一等高并发场景中，Redis缓存与MySQL数据库的**最终一致性**是保障系统稳定的关键。由于网络抖动、服务宕机等问题可能导致缓存更新失败，如何设计**补偿机制**兜底？

所以，这里尼恩给大家做一下系统化、体系化的梳理，使得大家可以充分展示一下大家雄厚的 “技术肌肉”，**让面试官爱到 “不能自已、口水直流”**。也一并把这个题目以及参考答案，收入咱们的 《[尼恩Java面试宝典PDF](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247497474&idx=1&sn=54a7b194a72162e9f13695443eabe186&scene=21#wechat_redirect)》V173版本，供后面的小伙伴参考，提升大家的 3高 架构、设计、开发水平。

> 最新《尼恩 架构笔记》《尼恩高并发三部曲》《尼恩Java面试宝典》的PDF，请关注本公众号【技术自由圈】获取，后台回复：领电子书

此文为上下两篇文章，  尼恩带大家继续，挺进 120分，让面试官 口水直流。

  

文章目录：

  - 说在前面

  - 基础回顾：4大缓存一致性基础方案

    - 1：Read-Through（读穿透）

    - 2：Write-Through（写穿透）

    - 3：Write behind （异步缓存写入）

    - 4：Cache-Aside Pattern（旁路缓存模式）

  - 常用方案Cache-Aside如何保证双写的数据一致性？

  - 为什么是删除缓存，而不是更新缓存呢？

  - 策略一：先更数据库，再更缓存

  - 为什么是删除缓存而不是更新缓存呢？

  - 策略二：先删缓存，再更新数据库

  - 策略三：先更数据库，再删缓存 (黄金组合)

  - 策略四：延迟双删策略

    - 延迟双删 实现步骤

    - 延迟双删 代码示例

    - 延迟双删 优点

    - 延迟双删 缺点

  - 60分 (菜鸟级) 答案

  - 一致性方案，如何提高性能？答案是： 异步删除 / 逻辑删除

  - 策略五：高并发场景下的 逻辑删除/逻辑过期 策略

    - 什么是逻辑删除？

    - 什么是异步重建？

    - 数据查询流程‌：

    - ‌异步重建 流程‌：

    - ‌数据更新 流程‌：

    - 逻辑删除Demo

    - 逻辑删除‌优势‌

    - 逻辑删除‌注意事项‌

    - 逻辑删除‌适用场景‌

  - 高并发场景下的策略之2： 异步删除

  - 策略六：先更数据库，再基于内存队列删缓存

  - 策略七：基于消息队列删除缓存

  - 策略八：基于binlog+消息队列删除缓存（弱入侵）

  - 一致性延迟时间：三种 异步删除方案一致性延迟时间的对比分析

    - ‌1. 基于内存队列删除缓存‌

    - ‌2. 基于消息队列删除缓存‌

    - ‌3. 基于binlog+消息队列删除缓存‌

    - ‌三种缓存异步删除方案的  对比总结‌

  - 80分 答案  (高手级)

  - 删除缓存失败后的 补偿机制（/重试机制）

  - ‌异步删除缓存‌的工业级 防抖动容错 方案（ 最终一致性方案优化）架构设计‌

    - 1：阻塞队列异步删除‌

    - ‌2：第一级补偿： 延迟队列重试‌

      - ‌延迟任务（支持重试计数）对象设计

      - 延迟队列消费者逻辑（含重试控制）

    - ‌3：第二级补偿，消息队列持久化重试‌

    - ‌4：第三级补偿， 定时任务兜底比对‌

      - 定时任务兜底比对‌的总体设计

      - 第一步：使用xxl-job进行 Redis key扫描实现：

      - 第二步：消息发送到Rocketmq：

      - 第三步：Flink数据比对和缓存操作：

  - ‌工业级 防抖动容错 方案（ 最终一致性 黄金组合方案 ）

    - 三级防御体系 最终一致性 黄金组合方案 ， 具体流程如下：

  - 可靠性保障 ：如何保障 新方案 无故障上线

  - 可靠性保障1 ：顺序消费

  - 可靠性保障2 ：灰度放量，新方案试点后再逐步放开

    - 1、流量染色策略‌

    - 2、灰度环境准备

    - 3、灰度放量流程

    - 4、灰度放量的监控与调整

    - 5、回滚与异常处理

    - 6、注意事项

  - 从CAP视角分析DB与Cache的数据一致性

  - 120分殿堂答案 (塔尖级)：

  - 尼恩架构团队塔尖的redis 面试题

  - 遇到问题，找老架构师取经

  

## 基础回顾：4大缓存一致性基础方案

Read-Through 和 Write-Through 是两种与缓存相关的策略，它们主要用于缓存系统与持久化存储之间的数据交互，旨在确保缓存与底层数据存储的一致性。

### 1：Read-Through（读穿透）

Read-Through 是一种在缓存中找不到数据时，自动从持久化存储中加载数据并回填到缓存中的策略。

具体执行流程如下：

- 客户端client 发起读请求到 CacheLayer缓存系统 。
- CacheLayer缓存系统检查是否存在请求的数据。
- 如果数据不在CacheLayer缓存中，CacheLayer缓存系统会透明地向底层数据存储（如Mysql）发起读请求。
- 数据库返回数据后，CacheLayer 缓存系统将数据存储到redis  缓存中，并将数据返回给客户端client 。
- 下次同样的读请求就可以直接从缓存中获取数据，提高了读取效率。

## ![image.png](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJMEFJ4bcO9PqibBSPFsMokaxic5Gjv4LF8CHJ7nP4tiaaxNTGRHSdDWtow/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

整体简要流程类似`Cache Aside Pattern`，但在缓存未命中的情况下，Read-Through 策略会自动隐式地从数据库加载数据并填充到缓存中，而无需应用程序显式地进行数据库查询。

Cache Aside Pattern 更多地依赖于应用程序自己来管理缓存与数据库之间的数据流动，包括缓存填充、失效和更新。而Read-Through Pattern 则是在缓存系统内部实现了一个更加自动化的过程，使得应用程序无需关心数据是从缓存还是数据库中获取，以及如何保持两者的一致性。

在Read-Through 中，缓存系统承担了更多的职责，实现了更紧密的缓存与数据库集成，从而简化了应用程序的设计和实现。

### 2：Write-Through（写穿透）

Write-Through 是一种在缓存中更新数据时，同时将更新操作同步到持久化存储的策略。

Write-Through（写穿透） 具体流程如下：

- 当客户端向缓存系统发出  Write 写请求时，缓存系统首先更新缓存中的数据。
- 同时，缓存系统还会把这次更新操作同步到底层数据存储（如数据库）。
- 当数据在数据库中成功更新后，整个写操作才算完成。
- 这样，无论是从缓存还是直接从数据库读取，都能得到最新一致的数据。

![image.png](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJiaibo3MPz8ic1Ssic3PR5k6WfLEqibNibv5Olo9mHE1NwLMiaIPFjRMb23QFw/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Read-Through 和 Write-Through 的共同目标是确保缓存与底层数据存储之间的一致性，并通过自动化的方式隐藏了缓存与持久化存储之间的交互细节，简化了客户端的处理逻辑。

这两种策略经常一起使用，以提供无缝且一致的数据访问体验，特别适用于那些对数据一致性要求较高的应用场景。

> 需要注意的是，虽然它们有助于提高数据一致性，但在高并发或网络不稳定的情况下，仍然需要考虑并发控制和事务处理等问题，以防止数据不一致的情况发生。

### 3：Write behind （异步缓存写入）

Write Behind（异步缓存写入），也称为 Write Back（回写）或 异步更新策略，是一种在处理缓存与持久化存储（如数据库）之间数据同步时的策略。

在这种模式下，当数据在缓存中被更新时，并非立即同步更新到数据库，而是将更新操作暂存起来，随后以异步的方式批量地将缓存中的更改写入持久化存储。

其流程如下：

- 应用程序首先在缓存中执行数据更新操作，而不是直接更新数据库。
- 缓存系统会将此次更新操作记录下来，暂存于一个队列（如日志文件或内存队列）中，而不是立刻同步到数据库。
- 在后台有一个独立的进程或线程定期（或者当队列积累到一定大小时）从暂存队列中取出更新操作，然后批量地将这些更改写入数据库。

![image.png](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJicnf3mIh0v1sRiaNrpPHhNB6PvrBSfYhibj0jj55tu2n7TE7P0GxBlMwA/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用 Write Behind 策略时，由于更新并非即时同步到数据库，所以在异步处理完成之前，如果缓存或系统出现故障，可能会丢失部分更新操作。

并且对于高度敏感且要求强一致性的数据，Write Behind 策略并不适用，因为它无法提供严格的事务性和实时一致性保证。

Write Behind 适用于那些可以容忍一定延迟的数据一致性场景，通过牺牲一定程度的一致性换取更高的系统性能和扩展性。

### 4：Cache-Aside Pattern（旁路缓存模式）

Cache-Aside Pattern（旁路缓存）模式，又叫旁路路由策略，在这种模式中，读取缓存、读取数据库和更新缓存的操作都是在应用程序中完成。

Cache Aside Pattern 是一种在分布式系统中广泛采用的缓存和数据库协同工作策略，在这个模式中，数据以数据库为主存储，缓存作为提升读取效率的辅助手段。

此模式是业务系统最常用的缓存策略。

旁路缓存又模式分为读缓存和写缓存。

- 先读缓存，缓存命中的话，直接返回数据；
- 如果缓存没有命中的话，就去读数据库，从数据库取出数据，放入缓存后，同时返回响应。

其工作流程如下：

![image.png](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJOTCbdPzic1rJfQ87WcIhBE8qplHpicoDEqjZEP9t8KOmAXY9T4WvDbmg/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

Cache-Aside Pattern（旁路缓存）模式读操作流程，具体如下：

- step 1：应用程序接收用户的数据查询的请求；
- step 2：应用程序优先从缓存查找数据；
- step 3：如果存在（cache hit），从缓存上查询出来，返回查询到数据；
- Step 4：如果不存在（cache miss），从数据库中查询数据并存入缓存中，返回查询到数据。

Cache-Aside Pattern（旁路缓存）模式写操作流程，具体如下：

- step 1：接收用户的数据写入的请求；
- step 2：先写DB；
- step 3：再删缓存。

数据什么时候从数据库（如Mysql集群）加载到缓存（如Redis集群）呢？

有以下两种加载模式可被选择：懒汉模式、饿汉模式。懒汉模式、饿汉模式可以理解为及时加载模式、延迟加载模式。

- 所谓懒汉模式，就会在使用时临时加载缓存。具体来说，就是当需要使用数据时，就从数据库中把它查询出来，然后写入缓存。第一次查询之后，后续的请求都能从缓存中查询到数据。
- 所谓饿汉模式，就是提前预加载缓存。具体来说，在项目启动的时候，预加载数据到缓存。当需要使用数据时，能直接从缓存获取数据，而不需要从数据获取。

饿汉模式，提前预加载数据到缓存的时机，能极大地提升请求处理的性能力，极大地提升系统的吞吐量。

此模式，适合于缓存那些不是经常变更的数据（例如商品类目数据），或者那些访问非常频繁的极热数据（例如秒杀商品数据）。

> 说   明
> 
> 懒汉模式、饿汉模式这组名词来自于Java的单例模式，关于Java的单例模式的详细介绍，请参考《Java高并发核心编程 卷2加强版》 （注意，是加强版）。

## 常用方案Cache-Aside如何保证双写的数据一致性？

Cache-Aside是日常开发中使用最多的缓存层高并发访问模式。

所以，面试官也喜欢围绕这种模式进行发问。

## 为什么是删除缓存，而不是更新缓存呢？

一个非常高频的问题是：Cache-Aside在写入的时候，为什么是删除缓存而不是更新缓存呢。

而且，很多大厂也喜欢问这个领域的问题，下面就是一道来自于社群的美团真题。

美团面试题

> Cache-Aside如何保证DB和Cache双写的数据一致性？

要完美的回答这个问题，咱们把Cache-Aside模式（旁路缓存模式）下的DB和Cache双写的策略，做一个系统化的梳理，大概分为如下五大策略。

- 策略一：先更数据库，再更缓存
- 策略二：先删缓存，再更新数据库
- 策略三：先更数据库，再删缓存
- 策略四：延迟双删策略
- 策略五：逻辑删除策略
- 策略六：先更数据库，再基于队列删缓存

如果能在面试的时候，把其中每一种策略的角色功能、适用场景、执行流程、优势弱点、改进策略进行系统化、体系化的陈述，无论是那个厂，无论是什么顶级的大厂，一定会对候选人的能力有十分的认可。

> 这里的内容，来自于《Java高并发核心编程 卷3加强版》
> 
> 有关6中策略的代码实操介绍，请参见 尼恩的 100Wqps三级缓存组件实操

## 策略一：先更数据库，再更缓存

在实际的业务场景中，一种常见的并发场景是：微服务Provider实例A、B同时进行同一个数据的更新操作。

按照先更数据库，再更缓存的策略，则微服务Provider实例A、B可能会出现下面的执行次序：

- step 1：微服务A去执行update DB
- step 2：微服务B去执行update DB
- step 3：微服务B去执行update Cache
- step 4：微服务A去执行update Cache

上面的执行流程，具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJaZrQnjiblmjodR3icyUdHlffryI5qv6gicWVIO5hlPVSsGbe23GDJ32tA/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图:先更数据库，再更缓存的并发执行案例

上面的执行流程，是典型的并发写入场景。

在图中的并发写入的场景中，Provider A进行数据的写入，Provider B也进行数据的写入。

最终的结果是：DB中的数据是Provider B的数据，Cache中的数据是Provider A的数据，出现DB和Cache数据不一致问题。

具体的原因是：Provider B的更新在Cache中的数据，被Provider A的更新在Cache中的数据覆盖了。

DB的更新次序先A后B，理论上Cache中的数据更新也应该是先A后B。

理论上，最终Cache中的数据应该是Provider B的数据，而不是Provider A的数据。

所以，在流程执行完毕后，缓存中的Provider A的数据为脏数据。

而之出现这个问题，是因为以上流程中step 3与step 4的执行均为操作缓存，都是高并发的操作，很难保证先后次序，所以缓存出现脏数据的概率很大。

## 为什么是删除缓存而不是更新缓存呢？

核心面试题

一个非常高频的问题是：Cache-Aside在写入的时候，为什么是删除缓存而不是更新缓存呢？

回到上一节的例子，在图中的并发写入的场景中，Provider A进行数据的写入，Provider B也进行数据的写入。

在这个例子中，写入DB的次序如下：

- Provider A先发起一个写操作，第一步先更新数据库
- Provider B再发起一个写操作，第二步更新了数据库

现在，由于分布式系统，无法保证并发操作的有序性，写入Cache的次序可能如下：

- Provider B先发起一个Cache写操作，第一步先更新Cache
- Provider A再发起一个Cache写操作，第二步更新了Cache

这时候，Cache保存的是Provider A的数据（老数据），DB保存的是B的数据（新数据），于是发生了DB和Cache数据不一致，Cache中出现脏数据。

如果使用删除操作取代更新操作，则Cache不会出现上面的脏数据问题。

具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJKBD0GXYc7ic6AiaL29ghlk2OeseF6PKG8l3ErJLpVhS2HWlwZBu7SsEA/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图:为何不更新缓存而是删除缓存

除了能够减少脏数据之外，更新缓存相对于删除缓存，还有两点劣势：

（1）如果写入Cache的值，是经过复杂计算才得到的话。更新缓存频率高的话，就会大大降低性能。

（2）及时更新缓存属于饿汉模式，适用于数据读取高频的场景。在写多读少的情况下，数据很多时候还没被读取到，又被更新了，这也浪费了Cache的空间，也降低了性能。

> 这里的内容，来自于《\[Java高并发核心编程 卷3加强版\]《Java高并发核心编程 卷3加强版》 （注意，是加强版）的  策略1
> 
> 策略1的代码实操介绍，请参见 尼恩的 100Wqps三级缓存组件实操

## 策略二：先删缓存，再更新数据库

在实际的业务场景中，一种常见的并发场景是：微服务Provider实例A进行数据的写入，而服务Provider实例 B同时进行同一个数据的读取操作。按照先删缓存，再更新数据库的策略，则微服务Provider实例A、B可能会出现下面的执行次序：

- step 1：微服务A去执行delete Cache
- step 2：微服务B去执行load from DB
- step 3：微服务B去执行update Cache
- step 4：微服务A去执行update DB

上面的执行流程，具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJ4ehvwmDu5K6ibeszxA23QkFuP5nsvNCM2XDbfyDVJ96gmwKbtmg3VIw/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图:先删缓存，再更新数据库的并发执行案例

上面的执行流程，是典型的并发读写场景。

在图中的并发读写的场景中，Provider A进行数据的写入，Provider B进行数据的查询。

最终，DB中的数据是Provider A的更新数据，Cache中的数据是Provider B从DB加载的数据，而这个数据已经过时，出现DB和Cache数据不一致问题。

具体的原因是：Provider B查询Cache的时候，Cache中的数据被删除，Provider B只能去DB查找，然后将数据更新在Cache。而Provider A在Provider B查完之后，竟然更新了DB，导致了DB和Cache的不一致。

出现这个DB和Cache的不一致问题的根本原因，大致如下：

写操作是先删Cache（操作1）再写DB（操作2），如果在此期间发生并发读，读取的动作很容易发生 操作1 和 操作2的中间，从而读取到过时的数据，最终导致Cache和DB不一致。

更为严重的时候，读操作把过期数据刷入Cache后，会导致后面比较长时间的不一致。

这个时间，一直持续到缓存过期，如说4个小时（以项目中的配置时间为准）。

上面的Cache和DB不一致，将导致一个严重的后面：后续的读取操作，都会使用Cache中的数据，所以，后面的读取操作都会使用过时数据。

> 这里的内容，来自于《\[Java高并发核心编程 卷3加强版\]《Java高并发核心编程 卷3加强版》 （注意，是加强版）的  策略2
> 
> 策略2的代码实操介绍，请参见 尼恩的 100Wqps三级缓存组件实操

## 策略三：先更数据库，再删缓存 (黄金组合)

先更数据库，再删缓存，基本上可以解决  "并发读写" 场景  Cache和DB数据不一致的问题。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJtYY0Hxzbtmiaict4JicDQBknOFNGS0WiaiblMiaJsJ1vsfqiaic5lc5GYRkuzA/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

但是，在一些特殊的场景中，还是会存在数据不一致的问题。

一种非常特殊的并发场景是：

微服务Provider实例A进行数据的写入操作，先写DB（操作1），再删Cache（操作2），如果由于某种原因出现了卡顿，没有及时把数据放入Cache。或者说，操作2发生了滞后。

此时，服务Provider实例 B进行一个数据的读取操作，读取的次序仍然是先读Cache，再读DB，很容易发生DB和Cache的不一致性。

按照先更数据库，再删缓存的策略，则微服务Provider实例A、B可能会出现下面的执行次序：

- step 1：微服务A去执行update DB
- step 2：微服务B去执行load from Cache
- step 3：微服务A去执行delete Cache，但是发生了延迟

上面的执行流程，具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJZb2lxfxH5bPLVBU0FNMAc7ichLiadibFO4l7MicDZ4qdfiayoiaibwmO0X0zw/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图:先更数据库，再删缓存的并发执行案例

在图中的并发读写的场景中，Provider A进行数据的写入，Provider B进行数据的查询。

微服务Provider实例A先写DB（操作1），再删Cache（操作2），如果 发生网络延迟 ，操作2严重滞后。

注意： 在操作2 完成之前，DB和Cache的数据，是不一致的。

在此期间，其他的数据读取操作，都会读取Cache中的过期数据，出现DB和Cache数据不一致问题。

总结不一致问题的根本原因，大致如下：

写操作是先写DB（操作1）再删Cache（操作2），如果在中间有 读请求要处理，读请求 从Cache读取到过时的数据，最终导致Cache和DB不一致。

等到写操作删除Cache（操作2）完成之后，Cache和DB的数据，会恢复一致性。

先更数据库，再删缓存 数据不一致的时间比较短， 一般就是一个网络抖动的 时间长度， 这个时间长度不会超过 1秒钟。

所以，策略三（先DB再Cache），比策略二（先Cache再DB）发生数据不一致的时间短。

相比较而言，推荐大家使用策略三，而不是策略二。

那么，策略三 也不是 完美无缺，具体 问题 是啥呢？

（1）写DB（操作1）和删Cache（操作2）之间，存在短时间的数据不一致；

（2）如果删Cache失败，存在较长时间的数据不一致，这个时间会一直持续到Cache过期；

如何解决策略三中Cache删除失败所导致的DB和Cache较长时间的数据不一致呢？

可以使用策略四：延迟双删。

> 这里的内容，来自于《\[Java高并发核心编程 卷3加强版\]《Java高并发核心编程 卷3加强版》 （注意，是加强版）的  策略3
> 
> 策略3的代码实操介绍，请参见 尼恩的 100Wqps三级缓存组件实操

## 策略四：延迟双删策略

什么是延迟双删呢？延迟双删是基于策略二进行改进，就是先删Cache，后写DB，最后延迟一定时间，再次删Cache。

在实际的业务场景中，一种常见的并发场景是：微服务Provider实例A进行数据的写入，而服务Provider实例 B同时进行同一个数据的读取操作。

按照先删Cache，后写DB，最后延迟一定时间，再次删Cache策略，则微服务Provider实例A、B可能会出现下面的执行次序：

- step 1：微服务A去执行delete Cache
- step 2：微服务B去执行load from DB
- step 3：微服务B去执行update Cache
- step 4：微服务A去执行update DB
- step 5：微服务A去执行 delay delete Cache

上面的执行流程，具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJp7NZjrdRFaa9ric9tumBY5oiaoUt3jtTLMwHY8JqRiar5rVsTIjfZXicSg/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

图：先删Cache，后写DB，再次延迟删Cache的并发执行案例

在图中的并发读写的场景中，Provider A进行数据的写入，Provider B进行数据的查询。

微服务Provider实例A先删Cache（操作1），再写DB（操作2），最后再二次延迟删除Cache（操作3）。

在操作2之前，如果有 并发  读请求，从DB读取到过时数据，可能出现DB和Cache数据不一致问题。

出现这个DB和Cache的不一致问题的根本原因，大致如下：

写操作是先删Cache（操作1）再写DB（操作2）， 读操作容易发生操作1、操作2的中间，从DB读到过时数据，最终导致Cache和DB不一致。

但是，这一轮的数据不一致，持续时间不会太长。

为啥呢？延迟双删 还有一个兜底的写操作：二次延迟删除Cache（操作3），从而保证数据一致。

所以，延迟双删也会存在数据不一致，不过是持续时间比较短而已。

### 延迟双删 实现步骤

**(1) 第一次删除缓存：**

在更新数据库之前，先删除缓存中的相关数据。这样可以避免在更新数据库期间，有其他线程读取到旧的缓存数据。

**(2) 更新数据库：**

执行数据库的更新操作，将新的数据写入数据库。

**(3) 延迟：**

等待一段时间（即延迟时间），这个时间需要根据业务场景和系统性能来确定，目的是确保在更新数据库后，所有可能读取旧数据的请求都已经完成。

**(4) 第二次删除缓存：**

经过延迟时间后，再次删除缓存中的相关数据。这一步是为了防止在延迟期间有其他线程将旧数据重新写入缓存。

### 延迟双删 代码示例

```
import redis.clients.jedis.Jedis;
import java.util.concurrent.TimeUnit;
public class CacheService {
    private Jedis jedis;
    private static final int DELAY_TIME = 100; // 延迟时间，单位：毫秒

    public CacheService() {
        this.jedis = new Jedis("localhost", 6379);
    }

    public void updateDataAndCache(String key, String newData) {
        try {
            // 第一次删除缓存
            jedis.del(key);

            // 更新数据库
            updateDatabase(key, newData);

            // 延迟一段时间
            TimeUnit.MILLISECONDS.sleep(DELAY_TIME);

            // 第二次删除缓存
            jedis.del(key);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private void updateDatabase(String key, String newData) {
        // 模拟更新数据库操作
        System.out.println("更新数据库，key: " + key + ", 新数据: " + newData);
    }

    public static void main(String[] args) {
        CacheService cacheService = new CacheService();
        cacheService.updateDataAndCache("user:1", "John Doe");
    }
}
```

### 延迟双删 优点

- 提高数据一致性：

通过两次删除缓存和中间的延迟操作，能够在一定程度上减少缓存和数据库数据不一致的情况，提高系统的数据一致性。

- 实现相对简单：

相比其他复杂的缓存一致性解决方案，延迟双删策略的实现较为简单，不需要引入过多的复杂逻辑和组件。

### 延迟双删 缺点

- 第一：延迟时间不好确定：

延迟时间的设置是一个难点.

如果设置过短，可能无法保证所有读取旧数据的请求都已经完成；如果设置过长，会影响系统的性能和响应时间。

- 第二： 多次删除性能低，会对Redis造成一定的压力

如果写操作比较频繁，可能会对Redis造成一定的压力；如何提高性能？ 是一个大的问题。

- 第三：如果第二次删除失败，会 长时间不一致性

极端情况下，DB和Cache存在较长时间的数据不一致，这个时间会一直持续到Cache过期，比如说4个小时（以 配置的缓存过期时间为准）。

如何  高可靠？ 也是是一个大的问题。需要提升第二次 删除的可靠性。

> 这里的内容，来自于《\[Java高并发核心编程 卷3加强版\]《Java高并发核心编程 卷3加强版》 （注意，是加强版）的  策略4
> 
> 策略4的代码实操介绍，请参见 尼恩的 100Wqps三级缓存组件实操

## 60分 (菜鸟级) 答案

尼恩提示，讲完 延迟双删  策略， 可以得到 60分了。

但是要直接拿到大厂offer，或者  offer 直提，需要 120分答案。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WMPawLFXelZEcgSo9xh2qibBqHa38xdvuDy4PWW7J5Zl7xiaLc92pJeAl2xL4f1GIkQ2RAuJ8VGJmMg/640?from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

尼恩带大家继续，挺进 120分，让面试官 口水直流。

## 一致性方案，如何提高性能？答案是： 异步删除 / 逻辑删除

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJrNPsggZic3zr6wbs0NzfCp0IeNNVTPsiat5icdPicibUuFKibdug6EHpflqg/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在缓存一致性策略中，异步删除和逻辑删除是提高性能的有效方式，以下是具体分析：

**方案一：异步删除**

- **原理**：在更新数据库时，不立即删除缓存，而是将删除缓存的操作放入异步队列中，由专门的线程或进程在后台异步执行。这样可以避免因同步删除缓存而导致的数据库事务阻塞，减少数据库操作的响应时间。
- **优势**：能显著提高系统的并发处理能力。例如，在高并发的电商系统中，当大量订单同时生成或商品信息频繁更新时，采用异步删除缓存，数据库可以快速响应写入操作，而不必等待缓存删除完成，从而提高整体系统的吞吐量，让系统能够处理更多的并发请求。

**方案二：逻辑删除**

- **原理**：并不真正从缓存中删除数据，而是通过设置一个逻辑标志位来表示数据是否已被删除。当查询缓存时，先判断逻辑标志位，如果标志位表示数据已被删除，则不返回该数据，视为缓存中不存在该记录。
- **优势**：一方面，避免了实际删除操作带来的性能开销，尤其是在缓存数据量较大且删除操作频繁的情况下，逻辑删除不需要进行物理数据的删除和内存空间的重新整理，能有效提高缓存的访问性能。另一方面，对于一些可能会频繁恢复 “删除” 数据的场景，逻辑删除只需修改标志位，而无需重新从数据库加载数据到缓存，大大提高了数据恢复的效率。

不过，这两种策略也有各自的局限性。

- 异步删除可能会导致缓存数据在短时间内不一致，需要通过一些补偿机制或定期校验来解决；
- 逻辑删除则会使缓存中存在一些 “无效” 数据，占用一定的内存空间，需要定期进行清理或采用更复杂的缓存淘汰策略来管理。

## 策略五：高并发场景下的 逻辑删除/逻辑过期 策略

如何提高性能呢？  先看逻辑删除，再看异步删除。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJrNPsggZic3zr6wbs0NzfCp0IeNNVTPsiat5icdPicibUuFKibdug6EHpflqg/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 什么是逻辑删除？

逻辑代表什么，假的删除，不是真正的删除。而是空间换时间，设置一些额外的标志， 标识这个数据已经过期。

在缓存数据中添加字段（如 `logicExpireTime`），表示业务有效期。

例如，数据实际有效期为30分钟，`logicExpireTime` 设置为  30分钟，  但物理缓存设置为1小时（包含冗余时间）。

写入缓存时，设置逻辑过期时间（如 `logicExpireTime = 当前时间 + 30分钟`）和较长物理TTL。

和逻辑过期配套的动作，叫做  ： 异步重建。

### 什么是异步重建？

查询时若发现逻辑过期，触发异步线程更新数据，期间仍返回旧数据，避免缓存击穿。

查询的时候，检查 logicExpireTime ，如果发现到时间了，另外有一个缓存的重建线程，进行异步重建

更新的时候， 更改 逻辑过期时间 = 当前时间

### 数据查询流程‌：

读取缓存时，先检查logicExpireTime：

- ‌**未过期**‌：直接返回数据。
- ‌**已过期**‌：触发异步重建（如通过消息队列或线程池），并返回旧数据。

### ‌异步重建 流程‌：

- ‌**加锁防并发**‌：使用互斥锁（如Redis的SETNX），确保仅一个线程重建，避免数据库压力。
- ‌**更新缓存**‌：从数据库获取最新数据，更新缓存并重置 `logicExpireTime` 和物理TTL。

### ‌数据更新 流程‌：

业务数据变更时，直接更新缓存并重置逻辑过期时间，或标记为过期（`logicExpireTime = 当前时间`），强制下次查询触发重建。

**什么是物理删除？**

当然， 这个key不能永久存在，最终还是需要做 物理删除，真正的从redis 删除。

物理删除 依赖Redis的TTL机制，到期后自动删除数据。

物理删除  通常设置为逻辑过期时间+冗余时间（如逻辑30分钟+冗余30分钟=物理1小时）。

> 逻辑过期时间= 业务过期时间
> 
> 物理过期时间= 逻辑过期时间  +  高并发冗余时间

### 逻辑删除Demo

以下是一个基于Java的 ‌**逻辑过期缓存Demo**‌ ，使用Redis模拟缓存层，包含逻辑过期判断、异步刷新、互斥锁防止缓存击穿等核心逻辑：

```
import redis.clients.jedis.Jedis;
import java.util.concurrent.;
public class LogicalExpirationCacheDemo {

    // 模拟Redis客户端
    private static Jedis jedis = new Jedis("localhost", 6379);

    // 线程池用于异步刷新缓存
    private static ExecutorService executor = Executors.newFixedThreadPool(5);

    // 分布式锁Key前缀
    private static final String LOCK_PREFIX = "lock:";

    // 逻辑过期时间字段名
    private static final String LOGIC_EXPIRE_FIELD = "logicExpireTime";

    // 默认缓存物理过期时间（兜底）
    private static final int DEFAULT_PHYSICAL_TTL = 3600; // 1小时

    public static void main(String[] args) throws InterruptedException {
        // 测试逻辑
        String key = "product:1001";

        // 第一次查询：触发缓存重建
        System.out.println("第一次查询: " + getData(key));

        // 模拟缓存逻辑过期
        jedis.hset(key, LOGIC_EXPIRE_FIELD, String.valueOf(System.currentTimeMillis() - 1000));

        // 高并发查询（模拟多个线程同时请求过期数据）
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 查询结果: " + getData(key));
            }).start();
        }

        Thread.sleep(2000); // 等待异步刷新完成
        System.out.println("最终缓存内容: " + jedis.hgetAll(key));

        executor.shutdown();
        jedis.close();
    }

    /
     * 获取缓存数据（核心逻辑）
/
    public static String getData(String key) {
        // 1. 从缓存获取数据
        String cacheData = jedis.hget(key, "data");
        String logicExpireStr = jedis.hget(key, LOGIC_EXPIRE_FIELD);

        // 2. 缓存不存在则初始化
        if (cacheData == null) {
            return rebuildCache(key);
        }

        // 3. 检查逻辑过期时间
        long logicExpireTime = Long.parseLong(logicExpireStr);
        if (logicExpireTime <= System.currentTimeMillis()) {
            // 4. 异步刷新缓存
            executor.execute(() -> {
                // 获取分布式锁（防并发重建）
                String lockKey = LOCK_PREFIX + key;
                if (jedis.setnx(lockKey, "1") == 1) {
                    try {
                        System.out.println("---> 开始异步刷新缓存: " + key);
                        rebuildCache(key);
                    } finally {
                        jedis.del(lockKey); // 释放锁
                    }
                }
            });
        }

        return cacheData;
    }

    /*
     * 重建缓存（模拟数据库查询）
/
    private static String rebuildCache(String key) {
        try {
            // 模拟数据库查询耗时
            Thread.sleep(1000);

            // 生成新数据（这里模拟实际业务数据）
            String newData = "最新数据@" + System.currentTimeMillis();

            // 设置逻辑过期时间（30分钟后过期）
            long newLogicExpire = System.currentTimeMillis() + 30 * 60 * 1000;

            // 更新缓存（设置物理TTL兜底）
            jedis.hset(key, "data", newData);
            jedis.hset(key, LOGIC_EXPIRE_FIELD, String.valueOf(newLogicExpire));
            jedis.expire(key, DEFAULT_PHYSICAL_TTL);

            return newData;
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

**1：数据结构**‌：

```
//Redis Hash结构：
product:1001
  ├── data : "最新数据@1680000000000"  # 实际数据
  └── logicExpireTime : "1680000000000" # 逻辑过期时间戳
```

**2‌：执行流程**‌：

**首次查询**‌：

缓存不存在 → 同步重建缓存

‌**后续查询**‌

- 缓存存在且未过期 → 直接返回
- 缓存存在但已过期 → 异步刷新缓存（保证快速返回旧数据）
- 多个并发请求 → 只有第一个线程能获取锁进行重建

### 逻辑删除‌优势‌

- ‌**高可用性**‌：逻辑过期期间仍返回旧数据，避免缓存雪崩。
- ‌**减少延迟**‌：异步重建不阻塞用户请求。
- ‌**资源优化**‌：物理过期兜底，防止内存泄漏。

### 逻辑删除‌注意事项‌

- ‌**短暂不一致**‌：异步重建期间可能返回旧数据，需业务容忍。
- ‌**重建失败**‌：需重试机制或告警，确保数据最终一致。
- ‌**锁竞争**‌：分布式锁需设置合理超时，避免死锁。
- ‌**冗余时间设置**‌：根据业务峰值调整，平衡内存占用与重建压力。

---

### 逻辑删除‌适用场景‌

- ‌**高并发读场景**‌：如热点商品信息、秒杀活动。
- ‌**允许最终一致性**‌：如资讯类、非实时统计数据。
- ‌**频繁更新数据**‌：需结合主动更新策略减少重建延迟。

通过逻辑过期策略，能够在保证系统高可用的同时，有效降低数据库负载，适用于对一致性要求不苛刻的高并发场景。

> 这里在 写《Java高并发核心编程 卷3加强版》 （注意，是加强版）的  时候，没有介绍， 没有写逻辑删除策略
> 
> 策略5的代码实操介绍，请参见 尼恩的 100Wqps三级缓存组件实操

## 高并发场景下的策略之2： 异步删除

如何提高性能呢？   前面介绍了逻辑删除，接下来，再看异步删除。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJrNPsggZic3zr6wbs0NzfCp0IeNNVTPsiat5icdPicibUuFKibdug6EHpflqg/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

实质上，异步删除是基于策略三进行改进。

首先回顾一下策略三的问题？

（1）写DB（操作1）和删Cache（操作2）之间，存在短时间的数据不一致；

（2）如果删Cache失败，存在较长时间的数据不一致，这个时间会一直持续到Cache过期；

策略三 是同步操作模式。异步删除  是异步操作模式。

在操作次序，异步删除 模式 和策略三保持一致， 先写DB后删除Cache。

不同的是，异步删除  引入队列，把删Cache的操作加入队列，后台会有一个异步线程、或者进程去异步消费队列中的删除任务，去执行删Cache的操作。

通过异步模式，主线程写完DB后直接返回，不用等删除Cache的结果，从而大大提升了性能。

而且异步化之后，就可以进一步把 多种异步队列结合使用， 进行 多级补偿。

基于队列删缓存，可以细分为：

- 第1种细分的方案：基于内存队列删除缓存
- 第2种细分的方案：基于消息队列删除缓存
- 第3种细分的方案：基于binlog+消息队列删除缓存

## 策略六：先更数据库，再基于内存队列删缓存

由于同步重试删除在性能上会影响吞吐量，所以常通过引入消息队列，将删除失败的缓存对应的 `key` 放入消息队列中， 异步 删除。

此策略把删Cache的操作加入任务队列，后台会有一个异步线程去异步消费任务队列里面的删除任务，去执行删Cache的操作，如果缓存删除失败，可以重试多次，确保删除成功。

在实际的业务场景中，一种常见的并发场景是：微服务Provider实例A进行数据的写入，而服务Provider实例 B同时进行同一个数据的读取操作。

Provider实例A先写DB，然后将删Cache加入任务队列；Provider实例 B则是先读缓存，没有数据再读DB。微服务Provider实例A、B可能会出现下面的执行次序：

- step 1：微服务A去执行update DB
- step 2：微服务A将delete Cache操作进入任务队列
- step 3：微服务B去执行load from Cache
- step 4：消费线程从任务队列提取delete Cache操作，执行删除Cache的操作，直到删除成功。

上面的执行流程，具体如下图所示：![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJluhpGHDpm8p1d4BuYQI9UbMzgiaofUVVbSMj6DX8fSy90Je3UdpaQtw/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在图中的并发读写的场景中，Provider A进行数据的写入，Provider B进行数据的查询。

微服务Provider实例A先写DB（操作1），再将删Cache操作加入任务队列（操作2）。

在删除Cache操作真正执行完成之前，其他的数据读取操作，都会读取Cache中的过期数据，出现DB和Cache数据不一致问题。

但是这种不一致，是短暂的。

任务队列的消费线程，会异步执行删除Cache的任务，并且会不断重试确保成功，删除Cache之后，DB和Cache数据不一致问题就会得到解决。

> 说   明
> 
> 保存删除Cache任务的队列，建议使用阻塞队列。
> 
> 任务队列的消费线程，可参考Rocketmq源码中的ServiceThread异步服务线程，其设计思想和执行性能都非常优越。
> 
> 后面尼恩会通过视频，介绍一下基于队列删除缓存的实操。

策略六也会出现这个DB和Cache的不一致问题，尤其是如果写操作非常频繁，队列的任务比较多，可能消费会比较慢，导致DB和Cache的不一致的时间会延长。

在这种情况下，可以根据任务队列的拥塞程度，开启多个线程，提升并发执行的效率。

与策略四相比，策略六的优势是：

（1）在写操作比较频繁的场景，策略四有两次删Cache操作，可能会对Redis造成一定的压力；策略六只有一次删Cache操作，Redis压力小一半。

（2）策略四如果删Cache失败，没有引入重试策略；策略六会多次重试，确保删Cache成功，如果重试多次仍然不成功，可以执行运维预警。

（3）策略四将写DB、删Cache这两个操作耦合在了一起，没有很好的做到单一职责；策略六将写DB、删Cache两个操作解耦，模块职责更加单一。

那么，策略六的问题是啥呢？

（1）如果写操作非常频繁，队列的任务比较多，可能消费会比较慢；需要引入多线程机制，加快消费速度。

（2）程序复杂度成倍上升，引入消费线程、任务队列，并且还需要不断进行性能优化。

（3）内存队列是JVM进程的内部队列，如果JVM崩溃，内存队列没有来得及处理的Cache记录删除任务会丢失，这些数据的Cache记录和DB记录会长时间不一致。

## 策略七：基于消息队列删除缓存

在前面的第一种细分方案中，将删除Cache的任务保存在内存队列，并不是高可靠的。

为了保证高可靠的删除Cache记录，这里引入高可用的独立组件——Rocketmq消息队列。

需要注意的是，这里引入的RocketMq消息队列是高可用的类型消息队列，不是单节点的类型消息队列，从而保障消息记录的高可用，保障Cache的删除操作只要没有被执行成功，就不会丢失。

引入高可用RocketMq消息队列之后，执行双写操作的Provider A的操作流程，有小幅度的调整。

Provider A需要将删除Cache的操作，序列化成Rocketmq消息，然后写入高可用Rocketmq消息队列中间件即可。

然后，由专门的消费者（Cache Delete Consumer）进行消息的消费，根据消息内容执行Cache记录删除工作。

DB和Redis双写的场景下，Provider A先更数据库，后基于消息队列删缓存的并发执行案例的执行流程，具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJy4P4IHwy9Vmo98Mr0PsDPxacReRJWpl9HVzteFiackHnUyKLUa6wRzA/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

引入高可用的独立组件RocketMq消息队列之后，Provider A的写入逻辑变得很简单，删Cache的时候，只需要发送消息到RocketMq即可，大大简化了Provider A程序的写入逻辑。

只是为了保证消息的高可靠传递，这里Provider A在发送消息的时候，需要使用同步发送模式，而不能使用异步发送的模式。

在消息投递的环节，由RocketMq高可用组件的ACK机制保证消息的高可靠投递。

如果消息第一次消费失败，RocketMq会重复多次进行投递，确保消息被正常消费，如果一直不能被成功消费，在重复投递一定的次数之后（默认16次），消息会进入死信队列。

系统的监控程序会对死信队列进行监控，一旦发现死信消息，监控程序会进行运维告警，由运维人员解决最终的缓存删除问题。除非Redis集群崩溃，一般都不会出现这样的极端情况。

和基于内存队列删除缓存，基于消息队列删除缓存的方案的优势是：

- 增加了Cache删除的可靠性，避免了因JVM崩溃所导致的内存队列中的记录丢失的问题。

那么，Provider在执行DB和Cache双写时，能不能进一步减少双写的负担，将发送删除Cache消息的操作，从双写逻辑中剥离，交给其他的组件去完成呢？

答案是可以的。具体来说，就是使用基于基于binlog+消息队列去删除Cache的方案。

## 策略八：基于binlog+消息队列删除缓存（弱入侵）

3大基于队列的异步缓存删除，可以细分为：

- 第1种细分的方案：基于内存队列删除缓存
- 第2种细分的方案：基于消息队列删除缓存
- 第3种细分的方案：基于binlog+消息队列删除缓存

前两种都存在 代码入侵， 第3种不存在业务代码的入侵。

以Mysql为例，可以使用阿里的Canal中间件，采集在数据写入Mysql时生成的binlog日志，Canal将日志发送到RocketMq队列。

DB和Redis双写的场景下，Provider A先更数据库，后基于基于binlog+消息队列删除缓存的并发执行案例的执行流程，具体如下图所示：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJqs6Bq7eS9D5icEf5SBUjzuP3Y8V7ibscjfKIwWWqKKo1u0G9gwuOAzTQ/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在消费端，可以编写一个专门的消费者（Cache Delete Consumer）完成缓存binlog日志订阅，筛选出其中的更新类型log，解析之后进行对应Cache的删除操作，并且通过RocketMq队列ACK机制确认处理这条更新log，保证Cache删除能够得到最终的删除。

基于binlog+消息队列去删除Cache的方案的优势是（没有业务入侵）：

- 微服务Provider在执行DB和Cache双写时，只需要执行写入DB的操作就可以了，不需要写缓存操作的 代码，大大简化了微服务Provider的业务逻辑。
- Cache的删除工作已经完全被Canal、RocketMq、专门的消费者（Cache Delete Consumer）三者相互结合去接管了。

## 一致性延迟时间：三种 异步删除方案一致性延迟时间的对比分析

以下针对 ‌**内存队列、消息队列、binlog+消息队列**‌ 三种方案的延迟特性进行对比：

### ‌1. 基于内存队列删除缓存‌

**延迟范围**‌：‌

毫秒级‌（通常 <10ms）。

‌**延迟来源**‌：

- 内存队列本身为内存操作，无网络传输与磁盘I/O开销‌。
- 消费者线程直接处理队列任务，无中间组件依赖。

**适用场景**‌：

- 高并发瞬时故障恢复（如网络抖动）‌。
- 对延迟敏感但允许短暂数据不一致的业务（如秒杀库存缓存删除）‌。

### ‌2. 基于消息队列删除缓存‌

**延迟范围**‌：‌

毫秒至秒级（通常 100ms~2s）。

**延迟来源**‌：

- 消息队列需完成 ‌**持久化存储**‌（如Kafka落盘）和 ‌**网络传输**‌（生产者→Broker→消费者）‌。
- 消费者需处理消息反序列化、重试策略（如指数退避）‌。

**适用场景**‌：  - 跨服务解耦场景（如分布式系统间缓存同步）‌。  - 允许稍高延迟但需高可靠性的业务（如订单状态更新）‌。

### ‌3. 基于binlog+消息队列删除缓存‌

**延迟范围**‌：‌

秒级至分钟级‌（通常 1s~30s）。

**延迟来源**‌：

- ‌**数据库主从同步延迟**‌：Binlog从主库同步到从库存在延迟（尤其高负载时）‌。
- ‌**Binlog解析与分发**‌：需通过Canal等工具解析并投递到消息队列，增加处理链路‌。

**适用场景**‌

- 强数据一致性要求且业务与缓存更新逻辑解耦的场景（如用户账户余额同步）‌。
- 可接受分钟级最终一致性的低频更新场景（如商品分类信息变更）‌。

### ‌三种缓存异步删除方案的  对比总结‌

| **‌**方案**‌** | **‌**延迟水平**‌** | **‌**可靠性**‌** | **‌**适用场景**‌** |
| --- | --- | --- | --- |
| 内存队列 | 最低（毫秒）（通常 <10ms） | 较低 | 瞬时故障恢复、高并发低延迟场景‌ |
| 消息队列 | 中等（秒级）（通常 100ms~2s） | 高 | 分布式解耦、异步可靠删除‌ |
| Binlog+消息队列 | 最高（分钟）（通常 1s~30s） | 最高 | 强一致性、业务无侵入式同步‌ |

方案选型：

- ‌**延迟敏感型业务** ‌  优先选择内存队列，但需容忍短暂不一致风险‌ 。
- ‌**平衡型场景**‌   选择消息队列，兼顾可靠性与延迟‌ 。
- ‌**弱一致性场景**‌   接受更高延迟，采用Binlog+消息队列兜底‌ 。

## 80分 答案  (高手级)

尼恩提示，讲完   **三种 异步删除方案一致性延迟时间的对比分析**  ， 可以得到 80分了。

但是要直接拿到大厂offer，或者  offer 直提，需要 120分答案。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WMPawLFXelZEcgSo9xh2qibBaygvmNsgKl0ThA65neN7Z0adPHAJibXPaGB7MHwBGxRXvQ20ezZwYlg/640?from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

尼恩带大家继续，挺进 120分，让面试官 口水直流。

## 删除缓存失败后的 补偿机制（/重试机制）

由于同步重试删除在性能上会影响吞吐量，所以常通过引入消息队列，将删除失败的缓存对应的 `key` 放入消息队列中，在对应的消费者中获取删除失败的 `key` ，异步重试删除。

这种方法在实现上相对简单，但由于删除失败后， 需要进行 有效的补偿。

- **先DB后Cache**：避免「更新DB成功但删除Cache失败」导致永久不一致 -
- **非阻塞删除**：不等待Redis响应直接返回用户，通过补偿机制兜底。

## ‌异步删除缓存‌的工业级 防抖动容错 方案（ 最终一致性方案优化）架构设计‌

在 Cache-Aside （包括延时双删）中可能存在更新数据库成功，但存在  异步删除缓存失败的场景，

如果发生这种情况，那么便会导致缓存中的数据落后于数据库，产生数据的不一致的问题。

缓存删除失败之后，可以设计一个  三级补偿机制，包括 延迟队列、消息队列补偿，定时任务scan key 比对补偿 ，确保 缓存的操作，万无一失。

‌异步删除缓存‌的工业级 防抖动容错 方案（ 最终一致性方案优化）架构设计‌，就是 三级补偿，或者说三级防御（延迟补偿+异步补偿+定时兜底）：

- 第一级补偿  延迟队列
- 第二级补偿 消息队列补偿
- 第三级补偿 定时任务scan key 比对补偿

三级防御体系（延迟补偿+异步补偿+定时兜底），结合流式处理和大数据技术，构建了完整的缓存一致性保障机制

以下方案基于 ‌**阻塞队列 → 延迟队列 → 消息队列 → 定时任务**‌ 四级链路，实现高可靠、低延迟的缓存删除补偿机制。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJ4LeuJXm3QiaCIBc7kX91ImZtCshdKzw5iby9iaeKp4bhXyBDCZFO6fdiaA/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)‌

### 1：阻塞队列异步删除‌

异步删除‌的优势： 解耦DB更新与缓存删除，避免主线程阻塞，提高主线程的性能。

‌**实现方案**‌：

```
// 阻塞队列（容量根据业务调整）
private BlockingQueue<String> asyncDeleteQueue = new LinkedBlockingQueue<>(1000);
// 更新DB后提交任务
public void updateDB(Data data) {
    db.update(data);
    asyncDeleteQueue.offer("cache_key:" + data.getId()); // 非阻塞提交
}

// 独立消费者线程池（固定大小防OOM）
private ExecutorService consumerPool = Executors.newFixedThreadPool(4);

// 启动消费者线程
consumerPool.submit(() -> {
    while (true) {
        String key = asyncDeleteQueue.take();
        boolean success = redis.del(key);
        if (!success) {
            delayQueue.offer(new DelayedTask(key, 100)); // 进入延迟队列
        }
    }
});
```

### ‌2：第一级补偿： 延迟队列重试‌

延迟一下， 应对瞬时故障（如网络抖动），

稍微延迟，减少无效重试次数。

设计一个 `DelayedTask` ， 增加‌**重试次数**‌属性 retryCount，并根据次数动态调整延迟时间，重试三次后转入消息队列。

在消费者处理失败时，检查retryCount，若未满3次， 增加retryCount，调整延迟时间，重新加入队列。

#### ‌延迟任务（支持重试计数）对象设计

```
public class DelayedTask implements Delayed {
    private final String key;
    private final int maxRetries; // 最大重试次数
    private int retryCount;       // 当前重试次数
    private long executeTime;     // 执行时间戳
    public DelayedTask(String key, int maxRetries, long initialDelayMs) {
        this.key = key;
        this.maxRetries = maxRetries;
        this.retryCount = 0;
        this.executeTime = System.currentTimeMillis() + initialDelayMs;
    }

    // 递增重试次数并更新执行时间
    public DelayedTask nextRetry() {
        this.retryCount++;
        long nextDelayMs = 100 * this.retryCount; // 延迟策略: 100ms, 200ms, 300ms
        this.executeTime = System.currentTimeMillis() + nextDelayMs;
        return this;
    }

    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(executeTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    @Override
    public int compareTo(Delayed o) {
        return Long.compare(this.executeTime, ((DelayedTask) o).executeTime);
    }

    // Getters
    public String getKey() { return key; }
    public int getRetryCount() { return retryCount; }
    public int getMaxRetries() { return maxRetries; }
}
```

#### 延迟队列消费者逻辑（含重试控制）

```
// 初始化延迟队列与消费者
private DelayQueue<DelayedTask> delayQueue = new DelayQueue<>();
private void startDelayQueueConsumer() {
    Executors.newSingleThreadExecutor().submit(() -> {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                DelayedTask task = delayQueue.take();
                boolean success = redis.del(task.getKey());

                if (!success && task.getRetryCount() < task.getMaxRetries()) {
                    // 生成下一次重试任务（递增延迟）
                    DelayedTask nextRetryTask = task.nextRetry();
                    delayQueue.offer(nextRetryTask);
                    log.warn("Retry cache delete: key={}, retry={}/{}", 
                        task.getKey(), nextRetryTask.getRetryCount(), task.getMaxRetries());
                } else if (!success) {
                    // 重试三次失败后转消息队列
                    mqProducer.send("cache_retry_topic", task.getKey());
                    log.error("Move to MQ after 3 retries: key={}", task.getKey());
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    });
}
```

### ‌3：第二级补偿，消息队列持久化重试‌

‌**目标**‌：解决进程重启或长时间故障导致的数据丢失。‌

**实现方案**‌（以RocketMQ为例）：

```
// RocketMQ 消费者类，利用原生重试机制
public class CacheRetryMQConsumer implements MessageListenerConcurrently {
    private final RedisClient redisClient; // 依赖注入Redis客户端

    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
        List<MessageExt> messages, 
        ConsumeConcurrentlyContext context
    ) {
        for (MessageExt message : messages) {
            String cacheKey = new String(message.getBody(), StandardCharsets.UTF_8);
            try {
                // 尝试删除缓存
                boolean success = redisClient.del(cacheKey);
                if (!success) {
                  // 最终失败后记录到数据库
        jdbcTemplate.update("INSERT INTO cache_fail_logs VALUES (?)", cacheKey);
                    // 删除失败，触发RocketMQ退避重试
                    return ConsumeConcurrentlyStatus.RECONSUME_LATER;
                }
            } catch (Exception e) {
                // 异常场景（如网络超时），触发重试
                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
        }
        // 消费成功
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
}
```

在消费端，当消费者处理消息抛出异常或返回ConsumeLater状态时，消息会被重新投递。

默认情况下，RocketMQ会为每个消息设置一个重试次数（通常为16次），每次重试的时间间隔逐渐增加，符合指数退避策略。

RocketMQ默认重试规则如下：

```
// RocketMQ 默认重试间隔（秒）
private int[] delayLevel = { 
  1, 5, 10, 30, 60, 120, 180, 240, 300, 360, 420, 480, 540, 600, 1200, 1800 
};
```

- 第1次重试：延迟1秒
- 第2次重试：延迟5秒
- ...
- 第16次重试：延迟1800秒（30分钟）
- ‌超过16次‌：消息进入死信队列（需提前配置）

### ‌4：第三级补偿， 定时任务兜底比对‌

基于xxl-job实现的第三级补偿机制的定时任务scan key比对补偿方向的详细设计：

#### 定时任务兜底比对‌的总体设计

- **任务触发**：

使用xxl-job定时任务调度器，设置合理的任务执行周期，如每30分钟-300分钟执行一次，具体周期可根据业务需求和数据更新频率来确定。

- **Redis key扫描**：

在任务执行时，使用Redis的`scan`命令对`hotkey:`前缀的key进行扫描。`scan`命令可以逐批遍历键空间，避免一次性加载过多数据导致内存问题。

- **消息发送到Rocketmq**：

将扫描到的Redis key封装成消息，发送到Rocketmq的消息队列中。消息内容可以包含key的名称、对应的业务数据标识等信息，以便后续处理。

- **Flink数据比对和缓存操作**：

Flink从Rocketmq中消费消息，获取Redis key后，从数据库中查询对应的原始数据，与Redis中的缓存数据进行比对。如果发现数据不一致，则根据业务逻辑更新缓存数据，确保缓存与数据库的一致性。

#### 第一步：使用xxl-job进行 Redis key扫描实现：

使用xxl-job定时任务调度器，设置合理的任务执行周期，如每30分钟-300分钟执行一次，具体周期可根据业务需求和数据更新频率来确定。

```
// 示例任务处理器伪代码
public class CacheKeyScanJobHandler extends IJobHandler {
    @Override
    public ReturnT<String> execute(String param) {
        // 获取分片参数
        int shardIndex = getShardIndex();
        int totalShards = getTotalShards();
        // 构造SCAN模式
        String pattern = "hotkey:"; 
        int batchSize = 500;

        // 分布式SCAN逻辑
        RedisConnection conn = getRedisConnection();
        ScanOptions options = ScanOptions.scanOptions()
                               .match(pattern)  //减少不必要的遍历*：
                               .count(batchSize)
                               .build();

        Cursor<byte[]> cursor = conn.scan(options);
        while(cursor.hasNext()){
            byte[] keyBytes = cursor.next();
            String key = new String(keyBytes);

            // 分片路由算法
            if(key.hashCode() % totalShards == shardIndex){

                sendToRocketMQ( new KeyMsg(key) ); // 发送到MQ
            }
        }
        return SUCCESS;
    }
}
```

上述代码展示了使用Jedis实现Redis的`scan`命令，对指定模式的key进行扫描，并将扫描到的key发送到消息队列。

`SCAN`命令是用于遍历 Redis 键空间的一种方式，它可以在不阻塞整个 Redis 服务器的情况下逐步遍历键。

然而，如前面所说，当键的数量非常庞大时，`SCAN`操作仍可能对性能产生一定影响，特别是如果执行不当，可能会导致短暂的卡顿或影响其他操作的响应时间。

在使用Redis的SCAN命令时，  可以通过下面的措施进行优化：

**(1) 合理设置COUNT参数：**

根据实际的键数量和服务器性能，选择一个合适的COUNT值。

一般建议在1000到10000之间进行测试，找到既能减少迭代次数，又不会对服务器造成过大压力的值。

**(2) 减少不必要的遍历：**

在实际应用中，尽量避免对整个数据库进行SCAN操作，而是通过MATCH参数指定特定模式的键，缩小遍历范围。

**(3) 分批处理数据：**

如果需要对SCAN命令获取的键进行进一步处理，可以将键分批发送到消息队列或进行其他操作，避免一次性处理大量数据导致的性能问题。

生产环境中可以使用`SCAN`，但需要综合考虑系统的具体情况和性能要求，通过合理的配置和监控来确保其不会对系统造成不良影响。

#### 第二步：消息发送到Rocketmq：

```
public static void sendToRocketMQ(KeyMsg  keyMsg) {
    // 获取Rocketmq生产者实例
    DefaultMQProducer producer = new DefaultMQProducer("producer_group");
    producer.setNamesrvAddr("rocketmq_namesrv_addr");
    try {
        producer.start();
        for (String key : keys) {
            Message msg = new Message("CacheCompareTopic", "Tag", keyMsg.getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(msg);
            // 处理发送结果
        }
    } catch (Exception e) {
        // 异常处理
    } finally {
        producer.shutdown();
    }
}
```

KeyMsg 类似的消息如下：

```
{
  "traceId": "补偿流水号",
  "keyType": "HOTKEY",
  "redisKey": "hotkey:user_123",
  "scanTime": "2023-09-01T12:00:00Z",
  "shardInfo": "2/5" // 分片标记
}

 将Redis key封装成消息并发送到Rocketmq，上面的代码中，指定了主题、标签和消息内容。

#### 第三步：Flink数据比对和缓存操作：

\`\`\`java
public static void compareAndCacheUpdate(DataStream<String> keyStream) {
    keyStream.map(key -> {
        // 从数据库查询原始数据
        String originalData = queryOriginalDataFromDB(key);
        // 从Redis获取缓存数据
        String cacheData = getCacheDataFromRedis(key);
        // 数据比对
        if (!originalData.equals(cacheData)) {
            // 更新缓存数据
            updateCacheDataToRedis(key, originalData);
        }
        return key;
    }).setParallelism(4); // 设置并行度
}
```

在Flink中，通过读取Rocketmq中的key消息流，进行数据比对和缓存更新操作，确保数据一致性。

## ‌工业级 防抖动容错 方案（ 最终一致性 黄金组合方案 ）

‌异步删除缓存‌的工业级 防抖动容错 方案（ 最终一致性方案优化）架构设计‌，就是 三级补偿，或者说三级防御（延迟补偿+异步补偿+定时兜底）：

- 第一级补偿  延迟队列
- 第二级补偿 消息队列补偿
- 第三级补偿 定时任务scan key 比对补偿

每一级别的延迟级别：

- 延迟队列（延迟50ms，指数退避, 10ms- 100ms毫秒级别）
- 消息队列（延迟重试15次，100ms-30秒 指数退避,  秒级  ）
- 定时任务（兜底扫描，  每30分钟-300分钟，小时级 ）

| ****层级**** | ****触发条件**** | ****实现方式**** | ****收敛时间**** |
| --- | --- | --- | --- |
| 内存队列 | Redis删除失败 | 阻塞队列+线程池 | 10ms- 100ms |
| 消息队列 | 内存队列重试失败 | RocketMQ事务消息 | 100ms-30秒 |
| 定时任务 | 定时触发/监控报警 | XXL-JOB 扫描关键的key | 30分钟-300分钟 |

三级防御体系 通过 ‌**阻塞队列异步化 + 延迟队列抗抖动 + 消息队列持久化 + 定时任务兜底**‌ 的四层架构，实现从「毫秒级」到「天级」的多维度补偿：

- ‌**99%场景**‌：通过阻塞队列和延迟队列在百毫秒内完成删除。
- ‌**0.99%场景**‌：依赖消息队列在秒级至分钟级恢复。
- ‌**0.01%极端场景**‌：由定时任务最终保障一致性。

### 三级防御体系 最终一致性 黄金组合方案 ， 具体流程如下：

三级防御体系（延迟补偿+异步补偿+定时兜底），结合流式处理和大数据技术，构建了完整的缓存一致性保障机制，兼顾可靠性（数据不丢）与实时性（秒级收敛）

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WNtgrnz938hx5I2V4yJbjiaJHapwGfHw5q0aRDGWJKfwIW1DL0jVJB6urr1bkdTOQtoVkcOZRBg2eQ/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

该方案适用于高并发和高可靠  都需要的要求苛刻的场景，比如，电商库存、金融交易等场景的 数据一致性。

## 附加的 可靠性保障 ：如何保障 新方案 无故障上线

- **消息有序**：Kafka /rocketmq 按照主键进行 分区路由 ，保证同一数据变更顺序
- **灰度放量**：新方案试点后再逐步放开：

## 附加可靠性保障方案1 ：顺序消费

Kafka /rocketmq 按照主键进行 分区路由 ，保证同一数据变更顺序

具体方案，请参见尼恩的文章：

[阿里面试：如何保证RocketMQ消息有序？如何解决RocketMQ消息积压？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247500158&idx=1&sn=483783e762febb12f46b60b99ae1180b&scene=21#wechat_redirect)

## 附加可靠性保障方案2 ：灰度放量，新方案试点后再逐步放开

没有经过线上验证的方案，不要全切，让风险可控，逐步放开

### 1、流量染色策略‌

- **随机百分比放量**：

通过为每个请求分配一个随机数来决定是否将其引导至新特性。适用于允许用户在新旧逻辑间交替的场景。

- **用户百分比放量**：

基于用户属性（如用户ID、地域等）进行哈希处理后分配百分比，确保同一用户在灰度期间始终访问新特性。

百分比分流‌：按用户ID哈希值进行10%流量切分‌

- **指定用户群体放量**：

将特定用户（如测试人员、特定城市的用户）直接引导至新特性，以便进行有针对性的测试或推广。

可以通过 请求头标记‌指定用户：通过X-Gray-Release: v2标识灰度流量‌可以通过 Cookie标识‌指定用户：对特定用户添加gray\_user=true标记‌

### 2、灰度环境准备

**(1) 部署 APISIX 及相关组件：**

按照官方文档，在 Kubernetes 集群中部署 APISIX、etcd 等组件，确保其稳定运行。

**(2) 创建灰度和基线上游服务：**

在 Kubernetes 中分别为新旧版本应用创建对应的 Service，如 `app-service`（基线）和 `app-service-gray`（灰度）。

**(3) 配置 APISIX 路由规则：**

通过 APISIX Dashboard 或 API，为应用配置路由规则，指定不同的匹配条件（如域名、请求参数等）将流量引导至相应的上游服务。

创建灰度上游服务

curl http://127.0.0.1:9180/apisix/admin/upstreams/100 \\-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \\-X PUT -d '{  "type": "roundrobin",  "nodes": {    "gray-service:8000": 1,    "prod-service:8000": 9  },  "labels": {"env": "gray"}}'

绑定路由规则

curl http://127.0.0.1:9180/apisix/admin/routes/1 \\-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \\-X PUT -d '{  "uri": "/api/\*",  "plugins": {    "traffic-split": {      "rules": \[        {          "match": \[            {"vars": \[\["http\_x-gray-release", "==", "v2"\]\]}          \],          "weighted\_upstreams": \[            {"upstream\_id": 100, "weight": 1}          \]        }      \]    }  },  "upstream\_id": "100"}'

### 3、灰度放量流程

**(1) 初始小流量测试：**

在灰度环境中，将一小部分流量（如 1%）引导至新特性，观察系统运行情况和用户反馈。

**(2) 逐步扩大放量比例：**

根据测试结果，逐步增加新特性的流量比例（如每次增加 10%），同时持续监控系统性能、错误率等指标。

**(3) 全量发布：**

当新特性在灰度环境中运行稳定且未发现重大问题后，将所有流量切换至新特性，完成全量发布。

### 4、灰度放量的监控与调整

- **实时监控**：

利用 APISIX 的可观测性功能，实时监控灰度放量过程中的流量分布、响应时间、错误率等关键指标。

- **问题处理**：

若在灰度放量过程中发现新特性存在问题，应及时回滚至基线版本，并对新特性进行修复和优化。

- **调整策略**：

根据监控数据和用户反馈，灵活调整灰度放量策略，如改变放量比例、调整用户群体范围等，以确保灰度放量的顺利进行。

### 5、回滚与异常处理

**(1) ‌自动回滚‌：**

当灰度环境错误率连续5分钟>3%时触发自动回滚‌

**(2) ‌手动干预‌：**

保留API接口用于紧急流量切换

```
//强制关闭灰度流量
curl -X PATCH http://apisix-admin:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' \
-d '{"plugins":{"traffic-split":{"disable":true}}}'
```

### 6、注意事项

- **配置管理**：将灰度放量的相关配置（如百分比、用户群体规则等）集中管理，便于动态调整和版本控制。
- **数据一致性**：在灰度放量过程中，要注意新旧特性之间的数据交互和一致性问题，避免因数据不同步导致的错误。
- **用户感知**：尽量减少灰度放量对用户感知的影响，确保用户在不知情的情况下平滑过渡到新特性。

通过以上基于 Apache APISIX 的灰度放量方案设计，可以有效地降低新特性发布带来的风险，确保系统的稳定性和可靠性，同时为用户提供更好的体验。

## 从CAP视角分析DB与Cache的数据一致性

CAP理论作为分布式系统的基础理论，它描述的是一个分布式系统在以下三个特性中：

- 一致性（Consistency）
- 可用性（Availability）
- 分区容错性（Partition tolerance）

分布式系统最多满足其中的两个特性：要么满足CA，要么CP，要么AP，无法同时满足CAP。

也就是说AP和CP是一组天敌，要满足AP高性能，只能舍弃CP。

在DB和Cache的分布式架构中，加入分布式Cache的目的是为了获得高性能、高吞吐，就是为了获得分布式系统的AP特性。

所以，如果需要数据库和缓存数据保持强一致（强CP特性），就不适合使用缓存。

从CAP的理论出发，使用缓存提升性能，就是会有数据更新的延迟，就会产生数据的不一致。

使用分布式Cache，可以通过一些方案优化，保证弱一致性，最终一致性的。

我们只能通过不断的方案迭代，减少不一致性的时间长度。

这需要Cache设计时：

- 结合业务仔细思考是否适合用缓存；
- 结合业务仔细思考缓存过期时间。

缓存一定要设置过期时间，这个时间太短、或者太长都不好。

- 如果过期时间太短，请求可能会比较多的落到数据库上，这也意味着失去了缓存的优势。
- 如果过期时间太长，缓存中的脏数据会使系统长时间处于一个延迟的状态，
- 而且，系统中长时间没有人访问的数据一直存在内存中不过期，浪费内存。

为啥DB和Cache没有办法强一致呢？

- 主要是写DB和删Cache是两个独立的操作，两个操作并没有保证原子性。
- 如果一定要强CP，就需要用分布式锁 保证写DB和删Cache两个操作的原子性，这里不是引入AP类型的普通Redis分布式锁，而是需要引入CP类型的Zookeeper分布式锁，或者引入CP类型的Redis RedLock，但是这种性能就非常，非常低了，不适用高并发场景。

## 120分殿堂答案 (塔尖级)：

尼恩提示，到了这里，讲完 动态库容、灰度切流 ， 可以得到 120分了。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/xlgvgPaib7WMPawLFXelZEcgSo9xh2qibBS7vx1QKibYa6diaTLXKus4UAibSwDOH0tG5yfeDD1P4R3uhZRiavOK3HBg/640?from=appmsg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

此文上一篇文章，  尼恩带大家继续，挺进 120分，让面试官 口水直流。

## 尼恩架构团队塔尖的redis 面试题

[京东面试： 亿级 数据黑名单 ，如何实现？（此文介绍了布隆过滤器、布谷鸟过滤器）](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247504154&idx=1&sn=f57f32c4d22ed6857cbf68d2bfbf4624&scene=21#wechat_redirect)

[希音面试：亿级用户 日活 月活，如何统计？（史上最强 HyperLogLog 解读）](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247504179&idx=1&sn=76e0985509626a1c72d61be1e2833c45&scene=21#wechat_redirect)

[史上最全： Redis: 缓存击穿、缓存穿透、缓存雪崩 ，如何彻底解决？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247503692&idx=1&sn=49c4a33f69f03dd814649413890476af&scene=21#wechat_redirect)

[史上最全：Redis脑裂 ，如何预防？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247503640&idx=1&sn=f3e820ced506bf4e46483d1acd2e0cba&scene=21#wechat_redirect)

[史上最全： Redis锁如何续期 ？Redis锁超时，任务没完怎么办？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247503521&idx=1&sn=e2f71881eb3abd52d484213089262ebc&scene=21#wechat_redirect)

[史上最全：Redis分布式 锁失效了，怎么办？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247503611&idx=1&sn=3cf05137d843f87757f3b7a3c76dbd7b&scene=21#wechat_redirect)

[史上最全：Redis分段锁，如何设计？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247501639&idx=1&sn=680711e2ea18ae925a5057a7ae15baa3&scene=21#wechat_redirect)

[史上最全： redis 锁的5个大坑，如何规避？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247503709&idx=1&sn=957520e8bd6e21b990b4ef40636d75e6&scene=21#wechat_redirect)

[史上最全：Redis热点Key，如何 彻底解决问题](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247503676&idx=1&sn=4201e57105f2c00127a8e1b50b17aa47&scene=21#wechat_redirect)

[史上最全：为啥Redis用哈希槽，不用一致性哈希？](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247501454&idx=1&sn=e44868bf5716a580980ccbcad525fc18&scene=21#wechat_redirect)

## 遇到问题，找老架构师取经

借助此文，尼恩给解密了一个高薪的 秘诀，大家可以 放手一试。**保证  屡试不爽，涨薪  100%-200%。**

后面，尼恩java面试宝典回录成视频， 给大家打造一套进大厂的塔尖视频。

通过这个问题的深度回答，可以充分展示一下大家雄厚的 “技术肌肉”，**让面试官爱到 “不能自已、口水直流”**，然后实现”offer直提”。

在面试之前，建议大家系统化的刷一波 5000页《[尼恩Java面试宝典PDF](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247497474&idx=1&sn=54a7b194a72162e9f13695443eabe186&scene=21#wechat_redirect)》，里边有大量的大厂真题、面试难题、架构难题。

很多小伙伴刷完后， 吊打面试官， 大厂横着走。

在刷题过程中，如果有啥问题，大家可以来 找 40岁老架构师尼恩交流。

另外，如果没有面试机会，**可以找尼恩来改简历、做帮扶。**

遇到职业难题，找老架构取经， 可以省去太多的折腾，省去太多的弯路。

尼恩指导了大量的小伙伴上岸，前段时间，**[刚指导一个40岁+被裁小伙伴，拿到了一个年薪100W的offer。](https://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486587&idx=2&sn=da85d7cacbdbc57403a9b901663df178&scene=21#wechat_redirect)**

狠狠卷，实现 “offer自由” 很容易的， 前段时间一个武汉的跟着尼恩卷了2年的小伙伴， 在极度严寒/痛苦被裁的环境下， offer拿到手软， 实现真正的 “offer自由” 。

    

       

## 空窗1年/空窗2年，如何通过一份绝世好简历，  起死回生  ？ 

**[空窗8月：中厂大龄34岁，被裁8月收一大厂offer， 年薪65W，转架构后逆天改命!](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486405&idx=1&sn=02336456cf4a09268c142c0b13327682&chksm=97b57a4da0c2f35be50007aee12b8f9e9aa7a74864f720846ae6dc58a5fa4b092e22d1805e1e&scene=21#wechat_redirect)**  

  

**[空窗2年：42岁被裁2年，天快塌了，急救1个月，拿到开发经理offer，起死回生](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486028&idx=1&sn=cf10ecda7a986b26e0359eba7a9cfeff&chksm=97b57bc4a0c2f2d2be01f2656ac65e9bd8854b91c64b01eb55f56e645f3c733525418808b06c&scene=21#wechat_redirect)**

  

**[空窗半年：35岁被裁6个月， 职业绝望，转架构急救上岸，DDD和3高项目太重要了](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486098&idx=1&sn=bbc5732b071477573bfab8a259d208d3&chksm=97b57b1aa0c2f20c27dd74b490b6062c9eec262a3ac25548534ff70290b172dcc51c6eafe532&scene=21#wechat_redirect)**

**[空窗1.5年：失业15个月，学习40天拿offer， 绝境翻盘，如何实现？](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486345&idx=1&sn=8978c2c98378e85efe9a089fa08fc9e8&chksm=97b57a01a0c2f317c29c7bedab1fa8a9f6101ab35cf005df447d662daa9c17d56f4e4c3a354a&scene=21#wechat_redirect)**

  

##  100W 年薪  大逆袭,  如何实现  ？ 

  

## **[100W案例，100W年薪的底层逻辑是什么？](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486315&idx=1&sn=52cc0a640c1499115f5a45b82097d857&chksm=97b57ae3a0c2f3f502293e84cde4bc892c45d06dee3579471a522ab497fb53e32640fb00c339&scene=21#wechat_redirect)** [如何实现年薪百万？ 如何远离  中年危机？](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486315&idx=1&sn=52cc0a640c1499115f5a45b82097d857&chksm=97b57ae3a0c2f3f502293e84cde4bc892c45d06dee3579471a522ab497fb53e32640fb00c339&scene=21#wechat_redirect)

**[100W案例2](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247485942&idx=1&sn=faa999fc9508435db7a88af2716492d9&chksm=97b5787ea0c2f168ac80d344670df16be12614d37161a893ebed88d4c915384fcbc024996403&scene=21#wechat_redirect)****[：](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247485942&idx=1&sn=faa999fc9508435db7a88af2716492d9&chksm=97b5787ea0c2f168ac80d344670df16be12614d37161a893ebed88d4c915384fcbc024996403&scene=21#wechat_redirect)**[40](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247485942&idx=1&sn=faa999fc9508435db7a88af2716492d9&chksm=97b5787ea0c2f168ac80d344670df16be12614d37161a893ebed88d4c915384fcbc024996403&scene=21#wechat_redirect)[岁小伙被裁6个月，猛卷3月拿100W年薪 ，秘诀：首席架构/总架构](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247485942&idx=1&sn=faa999fc9508435db7a88af2716492d9&chksm=97b5787ea0c2f168ac80d344670df16be12614d37161a893ebed88d4c915384fcbc024996403&scene=21#wechat_redirect)

**[环境太糟，如何升 P8级，年入100W？](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486369&idx=1&sn=0b32e8907dbcf26eaf33ea3f37ddd099&chksm=97b57a29a0c2f33f468d29dafe739204543873efb459f324871c3c68a3ecc178b1a9ab08a2fa&scene=21#wechat_redirect)**

## 如何  凭借  一份绝世好简历， 实现逆天改命，包含AI、大数据、golang、Java  等 

##   

- ## [逆天大涨：暴涨200%，29岁/7年/双非一本 ， 从13K涨到 37K ，如何做到的？](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486433&idx=1&sn=0f630e8fc7620b3bf0e054827b48d91e&chksm=97b57a69a0c2f37fad55b164ba7afb963e0db0366901da97a1fadc599cb5cf1ef95dfac736f2&scene=21#wechat_redirect)
- ## [逆天改命：27岁被裁2月，转P6降维攻击，2个月提 JD/PDD 两大offer，时来运转，人生翻盘!!  大逆袭!!](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486421&idx=1&sn=02a88c23c8fe689a662214d5297f88a0&chksm=97b57a5da0c2f34b1f87ed0bb948ddfc9327533bdfc44b88ad25201949bbdb8ae7439904e675&scene=21#wechat_redirect)
- ## [急救上岸：29岁（golang）被裁3月，转架构降维打击，收3个大厂offer， 年薪60W，逆天改命](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486392&idx=1&sn=16c34d7f960805f659adfb27676ee2de&chksm=97b57a30a0c2f326092cd7f79e125fd8bd5a006ed7459f1a2a81a8d557388297a02acd2cd5ce&scene=21#wechat_redirect)

- **[绝地逢生：](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486132&idx=1&sn=bb949c992b3a35935bf5fca57f19e2d8&chksm=97b57b3ca0c2f22aeb0e648191801aff933164dcf2b06eb4b6274df6499bffb0bba8e8169205&scene=21#wechat_redirect)**[9年经验](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486132&idx=1&sn=bb949c992b3a35935bf5fca57f19e2d8&chksm=97b57b3ca0c2f22aeb0e648191801aff933164dcf2b06eb4b6274df6499bffb0bba8e8169205&scene=21#wechat_redirect)[自考](http://mp.weixin.qq.com/s?__biz=MzIxMzYwODY3OQ==&mid=2247486132&idx=1&sn=bb949c992b3a35935bf5fca57f19e2d8&chksm=97b57b3ca0c2f22aeb0e648191801aff933164dcf2b06eb4b6274df6499bffb0bba8e8169205&scene=21#wechat_redirect)小伙伴，跟着尼恩狠卷3月硬核技术，面试机会爆表，2周后收3个offer ，满血复活

  

**职业救助站**

实现职业转型，极速上岸

  

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/9qY2Gt3PBs3icE6WcM4nBOdRRcczOW4wkNDgCMZhBfqeet9suwGHfgQduP33YYqqZLOppecMI72Pv5ibNkPeIk6g/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

关注**职业救助站**公众号，获取每天职业干货  
助您实现**职业转型、职业升级、极速上岸**  
\---------------------------------

**技术自由圈**

实现架构转型，再无中年危机

  

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/xlgvgPaib7WMtyFcJx45oIeWL1d6AENcdoAvuTPuq46E9n8pPe4SHXoSicuEVvhibHRSIzmEBPlC7POcDaqD6ID7w/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1&tp=webp)

关注**技术自由圈**公众号，获取每天技术千货  
一起成为牛逼的**未来超级架构师**

**几十篇架构笔记、5000页面试宝典、20个技术圣经  
请加尼恩个人微信 免费拿走**

**暗号，请在 公众号后台 发送消息：领电子书**

如有收获，请点击底部的"在看"和"赞"，谢谢

继续滑动看下一个

向上滑动看下一个

[知道了](https://mp.weixin.qq.com/s/)