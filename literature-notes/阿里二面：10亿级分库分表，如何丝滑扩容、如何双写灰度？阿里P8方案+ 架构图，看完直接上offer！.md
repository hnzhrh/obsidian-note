---
title: "阿里二面：10亿级分库分表，如何丝滑扩容、如何双写灰度？阿里P8方案+ 架构图，看完直接上offer！"
tags:
  - "clippings literature-note"
date: 2025-03-16
time: 2025-03-16T15:27:50+08:00
source: "https://mp.weixin.qq.com/s/Cj-v4k6kORjrfySfC1_wtA"
is_archie: "false"
---
原创 45岁老架构师尼恩 *2025年03月08日 13:49*

## 尼恩说在前面

在40岁老架构师 尼恩的**读者交流群**(50+)中，最近有小伙伴拿到了一线互联网企业如阿里、滴滴、极兔、有赞、希音、百度、网易、美团的面试资格，遇到很多很重要的面试题：

> 每天新增100w订单，如何的分库分表？
> 
> 10-100亿级数据，如何的实现 分库分表  的 丝滑扩容？

尼恩提示：

- 分库分表，是面试的核心重点。
- 分库分表，是面试的核心重点、核心重点。
- 分库分表，是面试的核心重点、核心重点、核心重点、核心重点。

所以，这里尼恩给大家做一下系统化、体系化的梳理，使得大家可以充分展示一下大家雄厚的 “技术肌肉”，**让面试官爱到 “不能自已、口水直流”**。也一并把这个题目以及参考答案，收入咱们的 《[尼恩Java面试宝典PDF](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247497474&idx=1&sn=54a7b194a72162e9f13695443eabe186&scene=21#wechat_redirect)》V173版本，供后面的小伙伴参考，提升大家的 3高 架构、设计、开发水平。

> 最新《尼恩 架构笔记》《尼恩高并发三部曲》《尼恩Java面试宝典》的PDF，请关注本公众号【技术自由圈】获取，后台回复：领电子书

此文为上下两篇文章，  尼恩带大家继续，挺进 120分，让面试官 口水直流。

本文的上一篇文章，

[阿里面试：每天新增100w订单，如何的分库分表？这份答案让我当场拿了offer](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247504991&idx=1&sn=4318f1e3f2f5db34c5599e9f73b50bc4&scene=21#wechat_redirect)

[  
](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247504991&idx=1&sn=4318f1e3f2f5db34c5599e9f73b50bc4&scene=21#wechat_redirect)

文章目录：

  - 尼恩说在前面

  - 尼恩解密：面试官的考察意图

    - 面试官的考察意图

  - 一、分库分表扩容 背景分析

  - 二、 数据增长预测

      - 短期趋势

      - 中期趋势

      - 长期趋势

  - 三、分库分表丝滑扩容 面临的问题与挑战

    - 迁移时间过长：

    - 准确性验证难：

    - 数据一致性问题：

    - 分布式事务问题：

    - 业务中断风险：

  - 四、分库分表丝滑扩容方案（核心：新旧双写+灰度切流+ 三级校验）

    - 1、数据层架构

    - 1.1 分片策略：

    - 1.2 数据迁移服务

    - 2、DAO层双写架构

    - 2.1 新旧双写模块

    - 2.2 自定义刚性事务管理器 / 刚柔结合的双事务架构

    - 3、中心控制层

    - 3.1 配置中心

    - 3.2 灰度开关

    - 5、定时任务

    - 6、理论小结

  - 五：实操1：数据双写同步

  - 六：自研事务管理器，实现 刚性事务双写

    - 6.1. ‌事务管理器定义

      - 2. ‌事务持有器封装‌

    - 6.2、关键实现细节

      - 1. ‌数据源配置与绑定‌

      - 2. ‌事务传播控制‌

    - 6.3、异常处理与优化

    - 6.4、验证与监控

      - 1. ‌事务一致性验证‌

      - 2. ‌监控指标‌

  - 七：刚柔结合事务控制策略 ，提高主事务写入的成功率

      - 1. ‌数据源定义与分片规则绑定‌

      - 2. ‌事务管理器实现细节‌

      - 3. ‌双写事务控制代码实现‌

    - 7.2、分布式事务协调与异常处理

      - 1. ‌柔性事务补偿机制‌

      - 2. ‌数据一致性校验工具‌

    - 7.4、性能优化与监控

      - 1. ‌连接池与批量写入优化‌

      - 2. ‌监控指标埋点‌

  - 八：Nacos动态控制 双写开关设计

    - 1. ‌Nacos配置项定义‌

    - 2. ‌动态配置监听与加载‌

    - 3. ‌双写逻辑改造‌

    - 4. ‌路由策略动态切换‌

    - 5、Nacos集成与运维管控

      - ‌5.1配置监听与事件响应‌

      - 5.2. ‌灰度发布与回滚策略‌

    - 8.6、验证与监控方案

      - 8.6.1. ‌双写开关生效验证‌

      - 8.6.2  ‌监控大盘设计‌

  - 九、读流量灰度：使用动态数据源实现 灰度流量切换

    - 9.1 动态数据源配置

    - 9. 2定义不同的分片规则

    - 9.3 DynamicDataSourceRouter‌ 数据源上下文管理‌和路由

      - 数据源  Type 信息 的设置与清理

      - 基于 Filter 实现灰度规则判断与清理（伪代码）

      - 关键实现要点说明

      - 总结  实现流程图

    - 9.5. 业务代码 如何 使用动态数据源  自动 灰度切流？

    - 9.6. 业务代码 如 控制器的实例展示

    - 注意事项

  - 十：数据校验与回滚

    - 10.1 校验服务（ 三级校验）

    - 10.2 自动修复

    - 10.3 监控告警

  - 120分殿堂答案 (塔尖级)：

  - 遇到问题，找老架构师取经

[  
](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247504991&idx=1&sn=4318f1e3f2f5db34c5599e9f73b50bc4&scene=21#wechat_redirect)

## 尼恩解密：面试官的考察意图

### 面试官的考察意图

**分库分表知识**：

考察面试者对分库分表概念、原理和策略的理解，包括水平分库、水平分表、垂直分库、垂直分表的适用场景和实现方式，看是否能针对 10 亿级数据量选择合适的分库分表方案。

**扩容技术**：

了解面试者对数据库扩容技术的掌握程度，如数据迁移、节点添加、负载均衡等，是否熟悉主流的扩容方法及其优缺点。

考察是否了解与分库分表和扩容相关的其他技术，如分布式事务处理、数据一致性保证、缓存策略等，以判断知识体系的完整性。

**复杂问题分析能力**：

10 亿级-10 0亿级数据的分库分表扩容是复杂问题，面试官想观察面试者能否清晰分析问题，考虑数据量、数据增长速度、业务场景、性能影响等多方面因素，提出合理的扩容思路和方案。

**应对挑战能力**：

扩容可能面临数据丢失、数据不一致、性能下降、业务中断等挑战，通过面试者的回答，了解其能否预见这些问题，并给出有效的应对措施，评估解决实际问题的能力。

## 一、分库分表扩容 背景分析

随着业务的不断发展，用户数量、业务数据量呈爆发式增长。

以电商平台为例，从初创时每日几百笔订单，发展到后来每日数十万甚至数百万笔订单，订单数据、用户数据、商品数据等不断累积，单个数据库表的数据量很快就会达到千万级甚至亿级。

当数据量达到一定规模后，数据库的读写性能会显著下降，数据的存储和管理也变得越来越困难

## 二、 数据增长预测

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

#### 短期趋势

在未来 1 - 2 年内，随着市场推广力度的加大、用户数量的持续增加以及业务拓展到新的领域，预计订单数据将以每年 30% - 50% 的速度增长。

这意味着：  1 - 2 年内 ，每天新增的订单数量可能在达到 130 万 - 150 万条。 订单总量 在 1亿级别。

#### 中期趋势

在 3 - 5 年的中期阶段，随着业务的稳定发展和市场份额的进一步扩大，订单数据的增长速度可能会有所放缓，但仍然会保持在每年 20% - 30% 的水平。

这意味着： 在 3 - 5 年内 ，每天新增订单数量可能接近 250 万条。订单总量 在 10亿级别。

#### 长期趋势

从 5 - 10 年的长期来看，考虑到市场的饱和以及竞争的加剧，订单数据的增长速度可能会逐渐稳定在每年 10% - 20% 左右。

这意味着： 在 5 - 10 年内 ，每天新增订单数量可能接近 600 万条。订单总量 在 100亿级别。

## 三、分库分表丝滑扩容 面临的问题与挑战

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

分库分表丝滑扩容面临诸多问题与挑战，主要体现在数据迁移、一致性保证、性能波动、业务影响 等方面，具体如下：

### 迁移时间过长：

对于 10 亿级数据，全量数据迁移耗时久，可能需要数天甚至数周时间。

期间若有新数据写入，还需不断进行增量数据迁移，增加了迁移的复杂性和时间成本。

### 准确性验证难：

大量数据迁移过程中，可能出现数据丢失、重复、错误等情况。

要确保迁移后数据与原数据完全一致，进行全面、准确的验证工作难度大，需要耗费大量的人力和计算资源。

### 数据一致性问题：

在数据迁移到新的库表过程中，既要保证原库表数据的正常读写，又要确保新库表数据的准确写入和同步，可能会出现数据在新旧库表之间不一致的情况。

例如，在迁移过程中发生网络故障、系统故障等，导致部分数据未成功迁移或迁移出现错误。

### 分布式事务问题：

扩容后，分布式事务涉及的节点和数据更多，协调和管理分布式事务的难度增大。不同数据库节点之间的事务一致性保证更加困难，可能出现部分节点事务提交成功，部分节点失败的情况，从而导致数据不一致。

### 业务中断风险：

在扩容过程中，尤其是进行数据迁移和切换时，可能需要暂停部分业务的读写操作，否则可能会导致数据不一致或数据丢失等问题。

即使采用一些灰度切换等策略，也很难完全避免对业务的短暂影响，如何在扩容过程中尽可能减少业务中断时间，是一个重要挑战。

## 四、分库分表丝滑扩容方案（核心：新旧双写+灰度切流+ 三级校验）

本方案旨在通过多维度多层次的设计，实现分库分表的丝滑扩容，确保系统在数据量和流量增长时能够平稳过渡，不影响业务的正常运行。

方案涵盖应用DAO层、控制层、数据层以及定时任务等多个方面，形成一个全面化、系统化的扩容架构。

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 1、数据层架构

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 1.1 分片策略：

老库分片1-N，新库分片1-2N

- 旧库按照既定的分片规则分为 N 个分片，
- 新库在旧库的基础上进行扩容，分为 2N 个分片，以应对未来数据量的增长。
- 确保分片键的选择合理，避免数据热点和分片不均的问题。
- ‌复合分片算法‌：采用`用户ID哈希+时间范围`双重路由策略，支持动态调整分片数量而不影响存量数据分布‌
- 一致性哈希环‌：虚拟节点数设置为物理节点的100倍，确保扩容后数据均衡‌

### 1.2 数据迁移服务

**全量数据迁移**：

- 使用高效的多线程并发 数据迁移工具 datax 或者kettle ，负责将旧库的数据全量迁移到新库。
- 在迁移过程中，使用迁移工具 datax 或者kettle  处理数据类型转换、字段映射等问题，确保数据的准确性和完整性。
- 采用`时间窗口分片法`，按创建时间顺序分批次迁移
- 同步偏移量双存储机制（Redis+MySQL），防止单点故障导致数据丢失‌

**增量数据同步**：

- 利用数据库的变更捕获技术（如 MySQL 的 Binlog），实时同步旧库的增量数据到新库。
- 基于`binlog+canal`实现实时增量同步，延迟控制在500ms内‌
- 确保在迁移期间，新库的数据能够与旧库保持同步更新。

**数据双写同步**：

- MySQL Binlog 同步到新库 ，存在数据延迟，导致新库的数据 可能不一定能立即可见。
- 可以采用 双写同步架构，保障 ，新库的数据能够与旧库保持同步更新。

### 2、DAO层双写架构

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 2.1 新旧双写模块

在扩容过程中，确保新旧数据架构的无缝切换，避免数据丢失或不一致。

**实现方式**：

- 在数据写入时，同时将数据写入旧库和新库，确保数据的同步性。
- 提供配置项，可根据业务需求灵活控制双写的开关，便于在扩容前后进行切换。

### 2.2 自定义刚性事务管理器 / 刚柔结合的双事务架构

**刚性事务管理器**：

- 针对关键业务操作，设计自定义的事务管理器，确保数据操作的原子性和一致性。
- 在新旧双写场景下，管理器需同时协调旧库和新库的事务，一旦任一库的操作失败，能够及时回滚所有相关操作。

**刚柔结合的双事务架构**：

- 对于非关键业务，采用柔性事务处理，允许一定程度的数据最终一致性，通过消息队列等方式异步处理数据同步。
- 结合刚性和柔性事务的优势，平衡系统的性能和数据一致性要求。
- ‌智能降级机制‌：在分片路由异常时自动切换至老库分片模式，保障服务可用性‌

### 3、中心控制层

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 3.1 配置中心

集中管理分库分表的相关配置，如分片规则、数据源信息等，实现配置的动态调整。

**实现细节**：

- 选用成熟的配置中心框架，如 Apollo、Nacos等，确保配置的高可用性和低延迟获取。
- 将分库分表的配置以键值对的形式存储，便于管理和修改。
- 提供配置的版本控制和回滚机制，防止因配置错误导致系统故障。

### 3.2 灰度开关

在扩容过程中，能够逐步放开流量，验证新架构的稳定性和正确性。

**实现方式**：

- 在系统中设置灰度开关，通过配置中心动态控制开关状态。
- 初始阶段，仅允许部分用户或请求访问新库，其余仍访问旧库。
- 根据监控数据和业务指标，逐步调整灰度开关，增加新库的流量占比，直至完全切换。
- 多维灰度策略‌：支持按用户ID段、业务类型、地理区域等多维度流量切换‌
- ‌流量染色机制‌：通过请求头标记区分新老数据流向，实现灰度发布‌
- **灰度切换**‌：按5%、20%、50%梯度逐步切换流量，每阶段观察12小时‌

### 5、定时任务

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

定期对新旧库的数据进行比对，确保数据的一致性。

### 6、理论小结

本分库分表丝滑扩容方案通过在应用DAO层、控制层、数据层以及定时任务等多方面的设计和实现，形成了一个全面化、系统化的扩容架构。

在实际的扩容过程中，需严格按照方案的步骤和要求进行操作，同时密切关注系统的运行状态和监控数据，及时处理可能出现的问题，确保扩容的顺利进行和业务的稳定运行。

接下来，讲究看看要点实操。

## 五：实操1：数据双写同步

在使用 Sharding-JDBC 进行数据双写同步前，需要配置两个数据源，分别对应旧库和新库。假设我们使用 Spring Boot 项目，在application.yml文件中进行如下配置：

```
spring:
  shardingsphere:
    datasource:
      names: oldDataSource,newDataSource
      oldDataSource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://old-db-host:3306/old_db?serverTimezone=UTC
        username: old_user
        password: old_password
      newDataSource:
        driver-class-name: com.mysql.cj.jdbc.Driver
        url: jdbc:mysql://new-db-host:3306/new_db?serverTimezone=UTC
        username: new_user
        password: new_password
    sharding:
      default-database-strategy:
        inline:
          sharding-column: user_id
          algorithm-expression: oldDataSource
      tables:
        your_table_name:
          actual-data-nodes: oldDataSource.your_table_name,newDataSource.your_table_name
```

上述配置定义了两个数据源oldDataSource和newDataSource，并通过 Sharding-JDBC 的分片策略，指定了表your\_table\_name在两个数据源中的实际数据节点。

```
@Transactional
public void doubleWrite(String sql, Object... args) {
    TransactionTypeHolder.set(TransactionType.XA);
    try {
        oldJdbcTemplate.update(sql, args);
        newJdbcTemplate.update(sql, args);
    } catch (Exception e) {
        // 记录异常日志
        log.error("数据双写失败", e);
        // 手动回滚事务
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    } finally {
        TransactionTypeHolder.clear();
    }
}
```

解下来就是 事务方案。

## 六：自研事务管理器，实现 刚性事务双写

第一种事务方案：自研事务管理器，实现 刚性事务双写

实现跨 Sharding-JDBC 数据源的统一事务管理，强制两个数据源在同一个事务边界内提交或回滚

### 6.1. ‌事务管理器定义

通过扩展 `AbstractPlatformTransactionManager` 实现跨 Sharding-JDBC 数据源的统一事务管理，强制两个数据源在同一个事务边界内提交或回滚‌。

```
public class DualShardingTransactionManager extends AbstractPlatformTransactionManager {  
    private final DataSource dataSourceOld;  // 旧库分片数据源  
    private final DataSource dataSourceNew;  // 新库分片数据源  
    @Override  
    protected Object doGetTransaction() {  
        // 绑定两个数据源的ConnectionHolder  
        return new DualTransactionHolder(  
            DataSourceUtils.getConnection(dataSourceOld),  
            DataSourceUtils.getConnection(dataSourceNew)  
        );  
    }  

    @Override  
    protected void doCommit(DefaultTransactionStatus status) {  
        DualTransactionHolder holder = (DualTransactionHolder) status.getTransaction();  
        try {  
            holder.getOldConnection().commit();  // 旧库提交  
            holder.getNewConnection().commit();  // 新库提交  
        } catch (SQLException e) {  
            throw new TransactionSystemException("双写提交失败", e);  
        }  
    }  

    @Override  
    protected void doRollback(DefaultTransactionStatus status) {  
        DualTransactionHolder holder = (DualTransactionHolder) status.getTransaction();  
        try {  
            holder.getOldConnection().rollback();  // 旧库回滚  
            holder.getNewConnection().rollback();  // 新库回滚  
        } catch (SQLException e) {  
            throw new TransactionSystemException("双写回滚失败", e);  
        }  
    }  
}
```

#### 2\. ‌事务持有器封装‌

```
private static class DualTransactionHolder {  
    private final Connection oldConnection;  
    private final Connection newConnection;  
    public DualTransactionHolder(Connection oldConn, Connection newConn) {  
        this.oldConnection = oldConn;  
        this.newConnection = newConn;  
    }  
}
```

### 6.2、关键实现细节

#### 1\. ‌数据源配置与绑定‌

```
// 旧库分片配置  
shardingOld:  
  dataSources:  
    ds_old:  
      url: jdbc:mysql://old-db:3306/db_old  
  sharding:  
    tables:  
      order:  
        actualDataNodes: ds_old.order  
//新库分片配置  
shardingNew:  
  dataSources:  
    ds0: jdbc:mysql://new-db0:3306/db_new  
    ds1: jdbc:mysql://new-db1:3306/db_new  
  sharding:  
    tables:  
      order:  
        actualDataNodes: ds$->{0..1}.order_$->{0..9}
```

#### 2\. ‌事务传播控制‌

在服务层通过 `@Transactional` 注解显式指定统一事务管理器：

```
@Service  
public class OrderService {  
    @Transactional(transactionManager = "dualShardingTransactionManager")  
    public void createOrder(Order order) {  
        // 旧库分片写入  
        try (Connection oldConn = DataSourceUtils.getConnection(shardingOldDataSource)) {  
            oldOrderMapper.insert(order);  
        }  
        // 新库分片写入  
        try (Connection newConn = DataSourceUtils.getConnection(shardingNewDataSource)) {  
            newOrderMapper.insert(order);  
        }  
    }  
}
```

‌**关键点**‌：

- 使用 `DataSourceUtils.getConnection()` 确保从统一事务上下文中获取连接‌
- 两个分片数据源的 SQL 操作需在同一线程内执行，避免跨线程事务失效‌

### 6.3、异常处理与优化

| **异常类型** | **处理方案** |
| --- | --- |
| ‌**单数据源提交失败**‌ | 强制回滚另一个数据源的事务，避免部分提交‌ |
| ‌**连接泄露**‌ | 通过 `DataSourceUtils.releaseConnection()` 在 finally 块中释放资源‌ |

### 6.4、验证与监控

#### 1\. ‌事务一致性验证‌

```
@Test  
public void testDualCommit() {  
    transactionTemplate.execute(status -> {  
        oldOrderMapper.insert(order);  
        newOrderMapper.insert(order);  
        return null;  
    });  
    // 验证新旧库数据一致性  
    Assert.assertEquals(  
        oldOrderMapper.selectById(order.getId()),  
        newOrderMapper.selectByShardingKey(order.getMemberId())  
    );  
}
```

#### 2\. ‌监控指标‌

| **指标** | **采集方式** |
| --- | --- |
| 双写事务平均耗时 | Micrometer 统计 `dualShardingTransactionManager` 提交耗时百分位‌ |
| 连接持有时间 | 日志记录 Connection 的 `getConnection()` 与 `close()` 时间戳差值‌ |

该方案通过自定义事务管理器实现跨 Sharding-JDBC 数据源的强一致性事务，适用于对数据一致性要求严苛的双写迁移场景‌ 。

新库如何写入失败， 会对业务产生影响。

不建议采用这种方式。

## 七：刚柔结合事务控制策略 ，提高主事务写入的成功率

在双写场景下，我们需要对主事务（旧库）和子事务（新库）采用不同的事务控制策略，以保证数据的一致性和系统的稳定性。

#### 1\. ‌数据源定义与分片规则绑定‌

‌**旧库数据源配置**‌：

application.yml

```
oldDataSource:  
  driver-class-name: com.mysql.jdbc.Driver  
  url: jdbc:mysql://old-db:3306/db_old  
  username: root  
  password: 123456
```

‌**新库分片数据源配置**‌：

```
shardingDataSource:  
  dataSources:  
    ds0:  
      driver-class-name: com.mysql.jdbc.Driver  
      url: jdbc:mysql://new-db0:3306/db_new  
      username: root  
      password: 123456  
    ds1:  
      driver-class-name: com.mysql.jdbc.Driver  
      url: jdbc:mysql://new-db1:3306/db_new  
      username: root  
      password: 123456  
  shardingRule:  
    tables:  
      order:  
        actualDataNodes: ds$->{0..1}.order_$->{0..9}  
        tableStrategy:  
          inline:  
            shardingColumn: member_id  
            algorithmExpression: order_$->{member_id % 10}  
        databaseStrategy:  
          inline:  
            shardingColumn: member_id  
            algorithmExpression: ds$->{member_id % 2}
```

‌**关键点**‌：

- 新库通过Sharding-JDBC定义分片规则，按`member_id`分库分表‌45
- 旧库保持单库单表结构，与新库分片规则需兼容（如`member_id`取模逻辑一致）‌58

#### 2\. ‌事务管理器实现细节‌

‌事务管理器定义‌：

```
@Configuration  
public class TransactionConfig {  
    // 旧库事务管理器  
    @Bean(name = "transactionManagerOld")  
    public PlatformTransactionManager oldTransactionManager(  
        @Qualifier("oldDataSource") DataSource dataSource) {  
        return new DataSourceTransactionManager(dataSource);  
    }  
    // 新库分片事务管理器（绑定ShardingSphere数据源）  
    @Bean(name = "transactionManagerSplit")  
    public PlatformTransactionManager splitTransactionManager(  
        @Qualifier("shardingDataSource") DataSource shardingDataSource) {  
        return new DataSourceTransactionManager(shardingDataSource);  
    }  
}
```

‌功能说明‌：

- `transactionManagerOld`管理旧库的本地事务，保证ACID特性‌
- `transactionManagerSplit`管理新库的分布式事务，支持跨分片操作‌

#### 3\. ‌双写事务控制代码实现‌

‌**服务层代码**‌：

```
@Service  
public class OrderService {  
    @Autowired  
    private OrderRepositoryOld oldRepository;  
    @Autowired  
    private OrderRepositoryNew newRepository;  
    @Transactional(  
        propagation = Propagation.REQUIRED,  
        transactionManager = "transactionManagerOld",  
        rollbackFor = Exception.class  
    )  
    public void createOrder(Order order) {  
        // 主事务：旧库写入（强一致性）  
        oldRepository.insert(order);  

        // 子事务：新库分片写入（独立事务）  
        writeToShardingDB(order);  
    }  

    @Transactional(  
        propagation = Propagation.REQUIRES_NEW,  
        transactionManager = "transactionManagerSplit",  
        noRollbackFor = {ShardingException.class}  
    )  
    private void writeToShardingDB(Order order) {  
        try {  
            newRepository.insert(order);  
        } catch (ShardingException e) {  
            // 分片路由失败时记录日志，不触发主事务回滚‌ 
            log.error("分片写入失败: orderId={}", order.getId());  
            mqSender.sendRetryMessage(order);  
        }  
    }  
}
```

‌**关键逻辑**‌：

- 主事务（旧库）使用`REQUIRED`传播，确保原子性‌
- 子事务（新库）使用`REQUIRES_NEW`传播，独立提交或回滚‌
- 新库写入异常时，通过MQ异步补偿保证最终一致性‌

### 7.2、分布式事务协调与异常处理

#### 1\. ‌柔性事务补偿机制‌

‌**补偿消费者实现**‌：

```
@Component  
@RocketMQMessageListener(topic = "ORDER_RETRY", consumerGroup = "retry-group")  
public class OrderRetryConsumer implements RocketMQListener<Order> {  
    @Override  
    public void onMessage(Order order) {  
        // 独立事务重试新库写入  
        TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManagerSplit);  
        transactionTemplate.execute(status -> {  
            newRepository.insert(order);  
            return true;  
        });  
    }  
}
```

‌**补偿策略**‌：

- 重试队列采用指数退避策略（如1s/5s/30s）‌
- 最大重试次数限制（如3次），超限后触发人工干预‌

#### 2\. ‌数据一致性校验工具‌

‌**校验逻辑示例**‌：

```
public void verifyData(Order order) {  
    // 旧库查询  
    Order oldOrder = oldRepository.selectById(order.getId());  
    // 新库分片查询（需计算分片路由）  
    String actualNode = shardingAlgorithm.getActualDataNode(order.getMemberId());  
    Order newOrder = newRepository.selectByShardingKey(actualNode, order.getId());  

    Assert.isTrue(oldOrder.equals(newOrder), "数据不一致");  
}
```

‌**校验维度**‌：

- 主键覆盖完整性（新旧库主键范围一致）‌
- 关键字段一致性（金额、状态等）‌

### 7.4、性能优化与监控

#### 1\. ‌连接池与批量写入优化‌

‌**连接池隔离配置**‌：

```
// 旧库使用Druid  
oldDataSource:  
  type: com.alibaba.druid.pool.DruidDataSource  
  initialSize: 5  
  maxActive: 20  
// 新库使用HikariCP  
shardingDataSource:  
  hikari:  
    maximum-pool-size: 50  
    connection-timeout: 5000
```

‌**批量插入优化**‌：

```
sqlCopy CodeINSERT INTO order_new_${shard} (id, member_id) VALUES (?, ?)  
ON DUPLICATE KEY UPDATE member_id=VALUES(member_id)
```

启用`rewriteBatchedStatements=true`提升批量性能‌

#### 2\. ‌监控指标埋点‌

| **指标** | **采集方式** | **告警规则** |
| --- | --- | --- |
| 双写事务TPS | 通过Micrometer统计`transactionManagerSplit`的提交频率‌ | 连续5分钟下降50%触发告警 |
| 分片路由失败率 | 日志分析ShardingSphere的`ShardingRouteException`‌ | 失败率 > 1%触发告警 |
| 异步补偿队列堆积量 | RocketMQ控制台监控`ORDER_RETRY`队列长度‌ | 堆积量 > 1000触发扩容 |

通过以上方案，可在Sharding-JDBC双写场景下实现高可靠的事务管理，结合刚性事务与柔性事务的优势，保障迁移过程的数据一致性与系统稳定性‌

## 八：Nacos动态控制 双写开关设计

注意我们需要提供一个动态开关，去控制开启和关闭新表的写入。

因为需求上线之后， 先同步一阶段 老的数据， 同步完毕之后开启同这个开关完成无缝对接。

另外，如果新库故障，也可以及时把这个开关关闭。

### 1\. ‌Nacos配置项定义‌

在Nacos中创建配置项 `dual-write-config`（Data ID），包含以下内容：

```
dualWrite:  
  enabled: true  # 双写开关  
  forceReadNew: false  # 强制读新库
```

‌**作用**‌：

- `enabled` 控制是否开启新库写入
- `forceReadNew` 用于灰度阶段强制查询新库（如数据校验完成后）‌

### 2\. ‌动态配置监听与加载‌

‌**配置类实现**‌：

```
@Configuration  
@RefreshScope  
public class DualWriteConfig {  
    @Value("${dualWrite.enabled:false}")  
    private AtomicBoolean enabled;  
    @Value("${dualWrite.forceReadNew:false}")  
    private AtomicBoolean forceReadNew;  
    // 提供线程安全的开关状态访问  
    public boolean isDualWriteEnabled() {  
        return enabled.get();  
    }  
    public boolean isForceReadNew() {  
        return forceReadNew.get();  
    }  
}
```

‌**关键点**‌：

- 使用 `@RefreshScope` 注解实现配置热更新‌
- `AtomicBoolean` 保证多线程环境下的状态一致性‌

### 3\. ‌双写逻辑改造‌

‌**服务层代码改造**‌：

```
@Service  
public class OrderService {  
    @Autowired  
    private DualWriteConfig dualWriteConfig;  
    @Transactional(transactionManager = "transactionManagerOld")  
    public void createOrder(Order order) {  
        // 旧库必写  
        oldRepository.insert(order);  

        // 根据开关判断是否写入新库  
        if (dualWriteConfig.isDualWriteEnabled()) {  
            writeToShardingDB(order);  
        }  
    }  

    @Transactional(propagation = Propagation.REQUIRES_NEW,  
                   transactionManager = "transactionManagerSplit")  
    private void writeToShardingDB(Order order) {  
        newRepository.insert(order);  
    }  
}
```

‌**优化点**‌：

- 双写开关生效时，主事务仅提交旧库，子事务异步处理新库写入‌
- 开关关闭后，通过增量同步工具（如Canal）补偿新库数据‌

### 4\. ‌路由策略动态切换‌

‌**查询路由适配**‌：

```
public Order getOrder(Long orderId) {  
    if (dualWriteConfig.isForceReadNew()) {  
        // 强制路由到新库分片  
        try (HintManager hintManager = HintManager.getInstance()) {  
            hintManager.setMasterRouteOnly();  
            return newRepository.selectByShardingKey(orderId);  
        }  
    } else {  
        return oldRepository.selectById(orderId);  
    }  
}
```

‌**说明**‌：

通过 `HintManager` 强制指定分片路由，用于灰度验证阶段‌

### 5、Nacos集成与运维管控

#### ‌5.1配置监听与事件响应‌

‌**Nacos监听器注册**‌：

```
@PostConstruct  
public void initNacosListener() {  
    ConfigService configService = NacosFactory.createConfigService(nacosProps);  
    configService.addListener("dual-write-config", "DEFAULT_GROUP", new AbstractListener() {  
        @Override  
        public void receiveConfigInfo(String configInfo) {  
            log.info("双写配置变更: {}", configInfo);  
            // 触发配置刷新（结合@RefreshScope）  
            refreshScope.refreshAll();  
        }  
    });  
}
```

‌**监控指标**‌：

- 配置变更次数（Nacos控制台）
- 双写开关状态（通过/metrics端点暴露）‌23

#### 5.2. ‌灰度发布与回滚策略‌

| **阶段** | **操作步骤** |
| --- | --- |
| ‌**灰度开启双写**‌ | 1\. Nacos修改`dualWrite.enabled=true` 2. 监控新库写入成功率与延迟‌ |
| ‌**全量切换**‌ | 1\. 开启`forceReadNew=true` 2. 停用增量同步工具‌ |
| ‌**异常回滚**‌ | 1\. 关闭双写开关 2. 启动反向同步工具回补旧库数据‌ |

### 8.6、验证与监控方案

#### 8.6.1. ‌双写开关生效验证‌

```
// 单元测试验证开关行为  
@Test  
public void testDualWriteSwitch() {  
    // 初始状态: 开关开启  
    orderService.createOrder(order);  
    assertExistInBothDB(order);  
    // 动态关闭开关  
    updateNacosConfig("dualWrite.enabled", "false");  
    orderService.createOrder(order2);  
    assertOnlyInOldDB(order2);  
}
```

#### 8.6.2  ‌监控大盘设计‌

| **监控项** | **采集方式** | **告警阈值** |
| --- | --- | --- |
| 双写开关状态 | 通过Spring Actuator暴露配置状态‌ | 状态异常持续1分钟 |
| 新库写入TPS | Micrometer统计`transactionManagerSplit`提交 | TPS下降50% |
| 增量同步延迟 | Canal监控Binlog消费延迟‌ | 延迟 > 60s |

通过Nacos动态控制双写开关，可在不重启服务的情况下灵活调整数据流向，结合事务管理器与路由策略，实现平滑迁移与快速故障恢复‌。

## 九、读流量灰度：使用动态数据源实现 灰度流量切换

为了在 Sharding-JDBC 中 实现灰度流量切换，这里 使用动态数据源。

动态数据源 借助 Spring 的 `AbstractRoutingDataSource` 来实现动态数据源的切换/灰度流量的灰度切流。

以下是具体的实现步骤

### 9.1 动态数据源配置

创建一个类来加载不同版本的 Sharding-JDBC 配置文件， 创建主库（旧规则）和灰度库（新规则）两个 Sharding-JDBC 数据源实例，

使用 `AbstractRoutingDataSource`  并创建 动态数据源 ， 实现动态路由，  通过 `@Bean` 注解注入 Spring 容器。

```
import org.apache.shardingsphere.api.config.sharding.ShardingRuleConfiguration;
import org.apache.shardingsphere.shardingjdbc.api.ShardingDataSourceFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import javax.sql.DataSource;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.Map;
import java.util.Properties;
// 使用 @Configuration 注解将该类标记为 Spring 配置类，Spring 会自动扫描并处理其中的 Bean 定义
@Configuration
public class DataSourceConfig {

    /

     * @throws SQLException 当创建数据源过程中出现 SQL 相关异常时抛出
     */
    @Bean(name = "oldDataSource")
    public DataSource oldDataSource() throws SQLException {
        // 调用 loadShardingRuleConfig 方法，从 "old-sharding-rule.yaml" 文件中加载旧版本的分片规则配置
        ShardingRuleConfiguration oldShardingRuleConfig = loadShardingRuleConfig("old-sharding-rule.yaml");
        // 使用 ShardingDataSourceFactory 创建旧版本的分片数据源，传入数据源映射、分片规则配置和属性对象
        return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), oldShardingRuleConfig, new Properties());
    }

    /
     * 创建并返回新版本的分片数据源 Bean

     * @return 新版本的分片数据源
     * @throws SQLException 当创建数据源过程中出现 SQL 相关异常时抛出
/
    @Bean(name = "newDataSource")
    public DataSource newDataSource() throws SQLException {
        // 调用 loadShardingRuleConfig 方法，从 "new-sharding-rule.yaml" 文件中加载新版本的分片规则配置
        ShardingRuleConfiguration newShardingRuleConfig = loadShardingRuleConfig("new-sharding-rule.yaml");
        // 使用 ShardingDataSourceFactory 创建新版本的分片数据源，传入数据源映射、分片规则配置和属性对象
        return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), newShardingRuleConfig, new Properties());
    }

    /
     * 创建并返回动态数据源 Bean，作为主要的数据源

     * @return 动态数据源
/
    @Bean
    @Primary
    public DataSource dynamicDataSource() {
        // 创建一个 HashMap 用于存储目标数据源，键为数据源名称，值为数据源对象
        Map<Object, Object> targetDataSources = new HashMap<>();
        try {
            // 将旧版本的分片数据源添加到目标数据源映射中，键为 "old"
            targetDataSources.put("old", oldDataSource());
            // 将新版本的分片数据源添加到目标数据源映射中，键为 "new"
            targetDataSources.put("new", newDataSource());
        } catch (SQLException e) {
            // 若创建数据源过程中出现 SQL 异常，打印异常堆栈信息
            e.printStackTrace();
        }
        // 创建动态数据源路由对象
        DynamicDataSourceRouter dataSource = new DynamicDataSourceRouter();
        // 设置动态数据源的目标数据源映射
        dataSource.setTargetDataSources(targetDataSources);
        try {
            // 设置动态数据源的默认目标数据源为旧版本的分片数据源
            dataSource.setDefaultTargetDataSource(oldDataSource());
        } catch (SQLException e) {
            // 若设置默认数据源过程中出现 SQL 异常，打印异常堆栈信息
            e.printStackTrace();
        }
        return dataSource;
    }

    /
     * 从指定的配置文件中加载分片规则配置

     * @param configFile 配置文件的名称
     * @return 分片规则配置对象
/
    private ShardingRuleConfiguration loadShardingRuleConfig(String configFile) {
        // 此处需要实现从配置文件加载 ShardingRuleConfiguration 的具体逻辑
        //  需要根据实际情况进行实现
        return null;
    }

}
```

### 9\. 2定义不同的分片规则

首先，需要为旧版本和新版本分别定义不同的 Sharding-JDBC 分片规则。

**旧版本规则示例（old-sharding-rule.yaml）**

```
dataSources:
  ds_0:
    url: jdbc:mysql://localhost:3306/ds_0
    username: root
    password: root
    driverClassName: com.mysql.cj.jdbc.Driver
shardingRule:
  tables:
    t_order:
      actualDataNodes: ds_0.t_order_$->{0..1}
      tableStrategy:
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order_$->{order_id % 2}
```

**新版本规则示例（new-sharding-rule.yaml）**

```
dataSources:
  ds_0:
    url: jdbc:mysql://localhost:3306/ds_0
    username: root
    password: root
    driverClassName: com.mysql.cj.jdbc.Driver
  ds_1:
    url: jdbc:mysql://localhost:3306/ds_1
    username: root
    password: root
    driverClassName: com.mysql.cj.jdbc.Driver
shardingRule:
  tables:
    t_order:
      actualDataNodes: ds_$->{0..1}.t_order_$->{0..1}
      tableStrategy:
        inline:
          shardingColumn: order_id
          algorithmExpression: t_order_$->{order_id % 2}
      databaseStrategy:
        inline:
          shardingColumn: user_id
          algorithmExpression: ds_$->{user_id % 2}
```

### 9.3 DynamicDataSourceRouter‌ 数据源上下文管理‌和路由

创建一个继承自 `AbstractRoutingDataSource` 的子类，用于根据一定的规则动态切换数据源。

```
public class DynamicDataSourceRouter extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DataSourceContextHolder.getDataSourceType();
    }
}
```

自定义 `DynamicDataSourceRouter` 继承 `AbstractRoutingDataSource`，

通过 DataSourceContextHolder工具类来保存和获取当前使用的数据源 Type 信息 。

DataSourceContextHolder工具类 的数据源  Type 信息  从 `ThreadLocal` 获取当前数据源标识。

```
public class DataSourceContextHolder {
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();

    public static void setDataSource(String dataSource) {
        contextHolder.set(dataSource);
    }

    public static String getDataSource() {
        return contextHolder.get();
    }

    public static void clearDataSource() {
        contextHolder.remove();
    }
}
```

#### 数据源  Type 信息 的设置与清理

下面的多数据源路由的例子中，DataSourceContextHolder的设置逻辑，如下

```
// 根据用户 ID 决定使用哪个数据源
        if (Integer.parseInt(userId) % 2 == 0) {
            DataSourceContextHolder.setDataSource("new");
        } else {
            DataSourceContextHolder.setDataSource("old");
        }
```

尼恩提示：如何在springboot的request 请求处理之前，通过filter设置，然后在请求处理完了之后，进行清除呢？

#### 基于 Filter 实现灰度规则判断与清理（伪代码）

```
@Component
public class DataSourceSwitchFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) 
        throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        try {
            // 1. 从请求头获取灰度标识（示例取user-id）
            String userId = httpRequest.getHeader("user-id");

            // 2. 执行灰度规则判断（示例：20%流量切到新库）
            if (StringUtils.isNotBlank(userId) && 
                Integer.parseInt(userId) % 100 < 20) {
                DataSourceContextHolder.setDataSource("new");
            } else {
                DataSourceContextHolder.setDataSource("old");
            }

            // 3. 继续执行后续请求处理
            chain.doFilter(request, response);
        } finally {
            // 4. 强制清理线程变量（关键步骤）
            DataSourceContextHolder.clearDataSource();
        }
    }
}
```

Spring Boot 配置类注册 Filter

```
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean<DataSourceSwitchFilter> registerFilter() {
        FilterRegistrationBean<DataSourceSwitchFilter> bean = new FilterRegistrationBean<>();
        bean.setFilter(new DataSourceSwitchFilter());
        bean.addUrlPatterns("/*"); // 拦截所有请求路径
        bean.setOrder(Ordered.HIGHEST_PRECEDENCE); // 最高优先级确保最先执行
        return bean;
    }
}
```

#### 关键实现要点说明

**1 灰度标识获取优化**‌

支持多维度参数判断，可扩展从 Cookie、Session 或 JWT 中提取灰度标记：

```
// 示例：从请求参数获取
   String grayFlag = httpRequest.getParameter("gray-version");
   // 示例：从认证信息获取（需结合 Spring Security）
   Authentication auth = SecurityContextHolder.getContext().getAuthentication();
```

**2 异常处理增强**‌添加灰度参数校验逻辑避免 NumberFormatException：

```
try {
       if (StringUtils.isNumeric(userId)) {
           // 执行模运算判断
       }
   } catch (Exception e) {
       log.error("灰度参数解析异常", e);
   }
<br>**3 动态规则扩展**‌<br>结合 Nacos/Apollo 实现动态调整灰度比例（需添加配置监听器）‌ ：<br><br>

 @RefreshScope
   @Configuration
   public class GrayRuleConfig {
       @Value("${gray.ratio:20}") 
       private int grayRatio; // 通过配置中心动态修改
   }
```

**4  流量染色验证**‌

在数据源切换后添加验证日志：

```
log.info("当前数据源: {}，灰度用户ID: {}", 
       DataSourceContextHolder.getDataSource(), userId);
<br>对数据源交叉污染现象进行检查。<br><br><br>**5 压力测试验证**‌<br>   使用 JMeter 模拟高并发请求，观察是否出现数据源交叉污染现象‌ <br><br>数据源交叉污染现象指在动态数据源切换场景中，请求错误地使用了非预期的数据源（如灰度请求误用主库，或主库请求误用灰度库），导致业务逻辑混乱或数据不一致。<br><br>数据源交叉污染 导致 灰度请求未正确路由到新规则数据源，仍访问旧规则库。<br><br>示例：灰度用户ID的请求本应路由到分库分表的新数据源，实际却落到未分库的主数据源。<br><br>关键原因是： 高并发下线程池复用线程时， 线程变量未及时清理。ThreadLocal中存储的数据源标识未在请求结束后清除，导致后续请求复用脏数据。<br><br>#### 总结  实现流程图<br><br>

textCopy CodeHTTP Request
    │
    ▼
[DataSourceSwitchFilter] → 解析灰度标识 → 设置ThreadLocal
    │
    ▼
[Controller] → 业务处理 → 通过AbstractRoutingDataSource自动路由‌:ml-citation{ref="3" data="citationList"}
    │
    ▼
[Response] ← 清理ThreadLocal（finally块保障）
```

该方案通过 Servlet Filter 实现了请求维度的数据源生命周期管理，确保每次请求结束后自动清理线程变量，避免内存泄漏和跨请求污染‌ 。

灰度场景，需要结合配置中心实现灰度规则动态调整‌ 。

### 9.5. 业务代码 如何 使用动态数据源  自动 灰度切流？

其实很简单了，业务代码是无感知的。

在业务代码中根据一定的规则（如用户 ID）来决定使用哪个数据源。

```
import org.springframework.stereotype.Service;
import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

@Service
public class BusinessService {

    @Autowired
    private DataSource routingDataSource;

    public void queryOrder(String userId) {
        // 根据用户 ID 决定使用哪个数据源
        try (Connection connection = routingDataSource.getConnection();
             PreparedStatement preparedStatement = connection.prepareStatement("SELECT * FROM t_order WHERE user_id = ?")) {
            preparedStatement.setString(1, userId);
            ResultSet resultSet = preparedStatement.executeQuery();
            while (resultSet.next()) {
                // 处理查询结果
            }
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            // 清除数据源上下文
            DataSourceContextHolder.clearDataSource();
        }
    }
}
```

### 9.6. 业务代码 如 控制器的实例展示

其实很简单了，业务代码是无感知的。

创建一个简单的控制器来测试灰度流量切换功能。

```
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
@RestController
public class OrderController {

    @Autowired
    private BusinessService businessService;

    @GetMapping("/orders/{userId}")
    public String queryOrder(@PathVariable String userId) {
        businessService.queryOrder(userId);
        return "Query order success";
    }
}
```

### 注意事项

- **异常处理**：

在实际应用中，需要对数据源创建、数据库操作等过程中的异常进行更完善的处理。

- **配置文件加载**：

`loadShardingRuleConfig` 和 `createDataSourceMap` 方法需要根据实际情况实现从配置文件加载 Sharding-JDBC 规则和数据源信息的逻辑。

- **线程安全**：

由于使用了 `ThreadLocal` 来保存数据源信息，要确保在多线程环境下不会出现数据混乱的问题。

通过以上步骤， 可以在 Sharding-JDBC 中实现基于动态数据源的灰度流量切换。

## 十：数据校验与回滚

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 10.1 校验服务（ 三级校验）

定期对新旧库的数据进行比对，确保数据的一致性。

**三级校验策略**‌：

- 字段级校验（小时级）

选取表中关键字段（如主键、时间戳、业务核心字段）作为比对基准，避免全字段比对带来的性能损耗‌

- 行级md5比对（每日低谷期）

通过`MD5()`函数生成哈希值（MySQL示例：`SELECT MD5(col_str) FROM table`）‌

- 统计指标校验（周级）‌

分片维度的 记录数

**实现细节**：

- 定时任务，定期按照数据的时间戳顺序，分批次对新旧库的数据进行比对。
- 对于比对发现的不一致数据，记录详细的信息，包括数据ID、字段差异等。

### 10.2 自动修复

**修复策略**：

- 对于数据比对发现的不一致情况，设计自动归档机制。

**修复流程**：

- 异常数据自动归档待人工核查‌。
- 执行修复操作，并记录修复日志，便于后续审计和问题追溯。

### 10.3 监控告警

**监控指标**：

- 设定关键的监控指标，如数据迁移进度、数据一致性比率、系统性能指标等。
- 通过监控工具实时采集这些指标数据，为系统的运行状态提供直观的展示。

**告警机制**：

- 当监控指标超出预设的阈值时，触发告警机制，通知相关人员及时处理。
- 告警方式可包括邮件、短信、即时通讯工具等多种渠道，确保告警信息能够及时送达。

**监控指标体系**

| **监控维度** | **核心指标** | **告警阈值** |
| --- | --- | --- |
| 事务一致性 | 双写成功率 | <99.9% (5分钟) |
| 数据完整性 | 校验差异率 | \>0.01% |
| 性能指标 | 分片P99延迟 | \>500ms |

## 120分殿堂答案 (塔尖级)：

尼恩提示，到了这里，讲完 动态库容、灰度切流 ， 可以得到 120分了。

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

此文上一篇文章，  尼恩带大家继续，挺进 120分，让面试官 口水直流。

[阿里面试：每天新增100w订单，如何的分库分表？这份答案让我当场拿了offer](https://mp.weixin.qq.com/s?__biz=MzkxNzIyMTM1NQ==&mid=2247504991&idx=1&sn=4318f1e3f2f5db34c5599e9f73b50bc4&scene=21#wechat_redirect)

## 遇到问题，找老架构师取经

借助此文，尼恩给解密了一个高薪的答题 秘诀，大家可以 放手一试。一定会 **屡试不爽，涨薪  100%-200%。**

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

  

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

关注**职业救助站**公众号，获取每天职业干货  
助您实现**职业转型、职业升级、极速上岸**  
\---------------------------------

**技术自由圈**

实现架构转型，再无中年危机

  

![图片](https://mp.weixin.qq.com/s/www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate(-249.000000,%20-126.000000)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

关注**技术自由圈**公众号，获取每天技术千货  
一起成为牛逼的**未来超级架构师**

**几十篇架构笔记、5000页面试宝典、20个技术圣经  
请加尼恩个人微信 免费拿走**

**暗号，请在 公众号后台 发送消息：领电子书**

如有收获，请点击底部的"在看"和"赞"，谢谢

继续滑动看下一个

向上滑动看下一个

[知道了](https://mp.weixin.qq.com/s/)