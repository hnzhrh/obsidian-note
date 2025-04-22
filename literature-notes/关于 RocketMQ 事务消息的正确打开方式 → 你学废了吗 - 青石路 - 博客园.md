---
title: "关于 RocketMQ 事务消息的正确打开方式 → 你学废了吗 - 青石路 - 博客园"
tags:
  - "clippings literature-note"
date: 2025-04-11
time: 2025-04-11T14:52:25+08:00
source: "https://www.cnblogs.com/youzhibing/p/15354713.html"
is_archie: false
---
## 开心一刻

　　昨晚和一哥们一起吃夜宵，点了几瓶啤酒

　　不一会天空下起了小雨，哥们突然道：糟了

　　我：怎么了

　　哥们：外面下雨了，我老婆还在等着我去接她

　　他给了自己一巴掌，说道：真他妈不是个东西

　　我心想：哥们真是个好丈夫

　　很快他补充道：喝酒怎么能分心呢

　　我一口啤酒直接笑喷而出

![](https://img2020.cnblogs.com/blog/747662/202110/747662-20211031140045749-1623899656.gif)

## 知识回顾

　　本文不讲什么是 RocketMQ ，不讲它的实现原理，只想和大家探讨下它的事务消息的正确使用方式

　　再探讨之前，先带大家回顾下知识点

### 事务消息的设计原理

RocketMQ  在 4.3.0 版中已经支持分布式事务消息，采用  2PC 的思想实现事务消息提交，同时增加一个补偿逻辑来处理二阶段超时或者失败的消息，如下图所示

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114152327998-2034372157.png)

　　什么，英文看不懂？贴心的我早已想到，中文版的也有

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114155453302-123012340.png)

　　其中有两个点：半事务、回查事务状态，值得我们重点回顾

### Half 消息

　　何谓 half 消息？

　　消息发送方把消息发送到 MQ 服务，但是此消息的状态被标记为不能投递，处于这种状态下的消息称为 half 消息；消费方不能消费 half 消息

　　发送方对 half 消息二次确认后，也就是 Commit 之后，消费方才可以消费到；如果是 Rollback，该消息则会被删除，永远不会被消费到

### 事务状态回查

　　如果在 RocketMQ 事务消息的二阶段过程中失败了，例如在做 Commit 操作时（上图中的第 4 步），出现网络问题导致 Commit 失败，那么需要通过一定的策略使这条消息最终被 Commit

　　RocketMQ 采用了一种补偿机制，称为“回查”。Broker 端对未确定状态的消息发起回查，将消息发送到对应的 Producer 端（同一个 Group 的 Producer），由 Producer 根据消息来检查本地事务的状态，进而执行 Commit 或者 Rollback

　　值得注意的是，RocketMQ 并不会无休止的的信息事务状态回查，默认回查 15 次，如果 15 次回查还是无法得知事务状态，RocketMQ 默认回滚该消息

　　更多细节请查看： [事务消息](https://github.com/apache/rocketmq/blob/master/docs/cn/design.md#5-%E4%BA%8B%E5%8A%A1%E6%B6%88%E6%81%AF)

## 实战示例

　　理论知识理解之后，就需要我们进行实操与分析了

### 需求背景

　　假设我们有两个服务：订单服务、积分服务，当用户成功下单之后，需要给用户加相应的积分

　　实现方式有很多种，你知道哪些？

　　假设我们用 RocketMQ 事务消息来保证最终一致性，我们又该如何实现？

### 环境准备

　　RocketMQ：4.8.0

　　rocketmq-client：4.9.2

　　Spring Boot：2.1.0.RELEASE

　　MySQL：5.7.29

　　MyBatis Plus：3.4.2

　　建表 SQL

```
-- order
CREATE TABLE \`order\`.\`t_order\` (
  \`order_id\` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  \`order_no\` char(20) NOT NULL COMMENT '订单号',
  \`user_id\` bigint(32) NOT NULL COMMENT '用户id',
  \`order_amount\` decimal(16,2) NOT NULL,
  \`note\` varchar(255) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (\`order_id\`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
-- 不一定非要存half消息的事务id，实现方式有很多，甚至可以不用这张表，直接通过 t_order 新增字段来实现
CREATE TABLE \`order\`.\`t_order_transaction_log\` (
  \`transaction_id\` varchar(32) NOT NULL COMMENT '主键（half 消息的事务id）',
  \`order_id\` bigint(20) NOT NULL COMMENT '订单主键',
  \`note\` varchar(500) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (\`transaction_id\`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- points
CREATE TABLE \`points\`.\`t_point\` (
  \`point_id\` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  \`order_no\` char(20) NOT NULL COMMENT '订单号',
  \`user_id\` bigint(20) NOT NULL COMMENT '用户id',
  \`point_num\` decimal(16,2) NOT NULL COMMENT '积分数量',
  \`note\` varchar(255) DEFAULT NULL COMMENT '备注',
  PRIMARY KEY (\`point_id\`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

View Code

　　项目地址： [spring-boot-rocketmq-order](https://gitee.com/youzhibing/qsl-project/tree/master/spring-boot-rocketmq-order) ， [spring-boot-rocketmq-points](https://gitee.com/youzhibing/qsl-project/tree/master/spring-boot-rocketmq-points)

　　后续只会对关键代码进行讲解，所以建议大家把代码 down 下来看看，保证有个基本的印象

　　回到标题，楼主为什么会强调：正确的打开方式

　　你猜对了，RocketMQ 事务消息的使用方式有很多种，楼主就结合工作项目中的使用方式，来和大家一起讨论下，哪些方式是正确的，哪些方式是不正确的（以及不正确的原因）

　　结合 Half 消息发送的时机，大致可分为三种：

![](https://img2020.cnblogs.com/blog/747662/202112/747662-20211201205233232-2046959564.png)

　　根据 half 消息的位置，我们暂且将这三种方式命名为：half 消息后置、half 消息中置、half 消息前置

　　我们逐个来讨论使用是否正确

### half 消息后置

　　这种方式有没有觉得似曾相识？与发普通消息是不是很类似？ 本地业务执行完之后，发普通消息给积分中心，是不是熟悉的味道？

　　但还是有区别的，至少有回查机制，我们结合伪代码具体看看

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114174246714-1857137004.png)

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114174742088-483105827.png)

　　我们来分析下各种异常情况，看看这种方式是否有问题

　　1、订单数据或订单事务日志落库异常，事务回滚，half 消息不会发送，没问题

　　2、half 消息发送异常，事务会回滚，没问题

　　3、half 消息发送未发生异常，但返回的不是 SEND\_OK 状态，代码抛出了异常，事务回滚，没问题

**思考** ：如果我们不关注 half 消息发送的结果，像这样

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114183448228-258705360.png)

　　　　最终，消息会推送给积分服务吗？

　　虽然看起来怪怪的，但又挑不出毛病

### half 消息中置

　　我们直接看伪代码

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114184604839-1251799697.png)

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114184623142-1286467764.png)

　　我们来分析下各种异常情况，看看这种方式是否有问题

　　1、订单数据落库异常，事务回滚，half 消息不会发送，没问题

　　2、half 消息发送异常，事务会回滚，没问题

　　3、half 消息发送未发生异常，但返回的不是 SEND\_OK 状态，代码抛出异常，事务会回滚，没问题

**思考** ：与之前的思考问题一样，如果我们不关注 half 消息发送的结果，最终消息会推送给积分服务吗？

　　　　只有发送 half 消息成功，并且发送状态为 SEND\_OK  ，才会执行  executeLocalTransaction  ，向  t\_order\_transaction\_log 表写入事务日志

　　　　那么即使 Broker  回查事务状态，它得到的结果始终是  UNKNOW ，最终 half 消息会被回滚，积分服务收不到消息

　　　　导致的问题就是： **用户下单成功，但却没有增加积分**

　　　　可见关注 half 消息发送结果的重要性

　　4、half 消息发送成功，且返回的是 SEND\_OK  状态，但  executeLocalTransaction 执行异常了，会是什么结果？

　　　　代码很明显，我们进行了 catch ，异常不会向上抛，订单落库还是成功的，只是订单事务日志落库失败了

　　　　返回 ROLLBACK\_MESSAGE ，half 消息会回滚，积分服务收不到消息

　　　　那么同样的问题又出现了： **用户下单成功，但却没有增加积分**

　　　　如果我们不 catch ，像这样

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114191907117-908074920.png)

　　　　理论上来讲，异常往上抛，订单数据会回滚， Broker  回查事务状态，一直返回  UNKNOW ，最终积分服务收不到消息

　　　　理论上来讲没问题，但事实呢？ 我们来实践一下

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114202018894-1287500828.gif)

哦豁，竟然没有打印异常日志，也就说异常被 catch 没有往外抛，订单数据也落库了

　　　　那么又会出现同样的问题： **用户下单成功，但却没有增加积分**

　　　　至于谁把异常 catch  了没往外抛，相信大家都能想到，这算是  rocketmq-client  的一个  bug ；源码稍后再跟，我们先看完前置

### half 消息前置

　　直接上伪代码

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114204436750-12713261.png)

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114220243470-1404794497.png)

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114211321194-1902071628.png)

　　我们来分析下各种异常情况，看看这种方式是否有问题

　　1、half 消息发送异常，本地事务不会执行，没问题

　　2、half 消息发送未发生异常，但返回的不是 SEND\_OK 状态，代码抛出异常，本地事务不会执行，没问题

**思考** ：与之前的思考问题一样，如果我们不关注 half 消息发送的结果，会是什么结果？

　　　　只有 half 消息发送成功，且返回状态是 SEND\_OK  才会执行  executeLocalTransaction

　　　　即使 Broker  回查事务状态，得到的结果始终是  UNKNOW ，最终 half 消息会被回滚，积分服务收不到消息

　　　　订单服务与积分服务都没有落库成功，也就说是没问题的

　　3、half 消息发送成功，且返回的状态是 SEND\_OK  ，但  executeLocalTransaction 执行异常了，会是什么结果

　　　　也就是 save 方法执行异常了，我们来实践下

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114221506087-1339192226.gif)

　　　　异常还是被 catch 了没往外抛，但是订单数据却回滚了，就结果而言是没问题的

　　　　half 消息发送成功了，但是 Broker  一直未收到本地事务的确认消息，  Broker  会回查，得到的结果始终是  UNKNOW ，最终 half 消息会被回滚，积分服务收不到消息

　　　　订单数据回滚了，积分服务未收到消息，那么此种情况是没问题的

　　看起来挺顺眼，异常情况下也没什么问题

## rocketmq-client 的 bug

　　需要弄清楚的问题有两个：

　　1、half 消息中置， executeLocalTransaction 的异常为什么没有抛出来

　　2、half 消息前置， 异常同样没有抛出来，为什么订单数据却回滚了

　　先看第一个问题，我们来跟下源码

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114223304772-638168160.gif)

rocketmq-client 捕获了异常，但并未向外抛

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114223533271-1522407907.png) 　　其实 RocketMQ 是有打印日志的，只是楼主的日志配置的不对，导致控制台未打印出来

　　对于第 1 个问题，相信大家已经清楚了

　　关于第 2 个问题，我就不具体分析了，我给个提示，从事务 AOP 的控制范围与异常抛出点来考虑，如下图

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114234100008-389477910.png)

![](https://img2020.cnblogs.com/blog/747662/202111/747662-20211114234113018-2010598350.png)

## 最终一致性

　　前面讲了那么多，都是讲的订单服务，总结起来就是：事务消息（而非 half 消息）发送成功，那么本地事务一定是执行成功的

　　保证的是事务消息的发送与订单服务的强一致

　　如果积分服务消费异常呢？

　　那对不起，RocketMQ 事务消息处理不了这种情况，回滚不了订单服务的数据，只能通过补偿机制（比如人工修复）修复积分服务的数据

## 总结

　　1、三种方式的抉择

　　　　half 消息中置，问题比较多，不推荐

　　　　half 消息后置，看起来挺别扭的（难道只是楼主这么觉得？），倒是没什么问题

　　　　half 消息前置，符合 RocketMQ 事务消息的设计原理，推荐采用此种方式

　　2、一定要关注 half 消息发送的结果，不抛异常不代表一定成功了，必要时需要根据 half 消息发送的结果做后续逻辑处理

　　3、最终一致性

　　　　RocketMQ 考虑的是数据最终一致性，上游服务提交之后，下游服务最终只能成功，做不到回滚上游服务的数据

## 参考

[基于RocketMQ分布式事务 - 完整示例](https://juejin.cn/post/6844904099993878536)

posted @ [青石路](https://www.cnblogs.com/youzhibing) 阅读( 2750 )  评论( 2 ) [编辑](https://i.cnblogs.com/EditPosts.aspx?postid=15354713) [收藏](https://www.cnblogs.com/youzhibing/p/) [举报](https://www.cnblogs.com/youzhibing/p/)

登录后才能查看或发表评论，立即 [登录](https://www.cnblogs.com/youzhibing/p/) 或者 [逛逛](https://www.cnblogs.com/) 博客园首页

[【推荐】100%开源！大型工业跨平台软件C++源码提供，建模，组态！](http://www.uccpsoft.com/index.htm)  
[【推荐】国内首个AI IDE，深度理解中文开发场景，立即下载体验Trae](https://www.trae.com.cn/?utm_source=advertising&utm_medium=cnblogs_ug_cpa&utm_term=hw_trae_cnblogs)  
[【推荐】编程新体验，更懂你的AI，立即体验豆包MarsCode编程助手](https://www.marscode.cn/?utm_source=advertising&utm_medium=cnblogs.com_ug_cpa&utm_term=hw_marscode_cnblogs&utm_content=home)  
[【推荐】轻量又高性能的 SSH 工具 IShell：AI 加持，快人一步](http://ishell.cc/)  
[![](https://img2024.cnblogs.com/blog/35695/202502/35695-20250225181349486-809978451.jpg)](https://www.marscode.cn/?utm_source=advertising&utm_medium=cnblogs.com_ug_cpa&utm_term=hw_marscode_cnblogs&utm_content=home)

**编辑推荐：**  
· [一文彻底搞懂 MCP：AI 大模型的标准化工具箱](https://www.cnblogs.com/BNTang/p/18815937)  
· [电商平台中订单未支付过期如何实现自动关单？](https://www.cnblogs.com/seven97-top/p/18810985)  
· [用.NET NativeAOT 构建完全 distroless 的静态链接应用](https://www.cnblogs.com/hez2010/p/18813775/dotnet-nativeaot-distroless-statically-linked-app)  
· [为什么构造函数需要尽可能的简单](https://www.cnblogs.com/CareySon/p/18801509)  
· [探秘 MySQL 索引底层原理，解锁数据库优化的关键密码(下)](https://www.cnblogs.com/ibigboy/p/18803885)  

**阅读排行：**  
· [短信接口被刷爆：我用Nginx临时止血](https://www.cnblogs.com/aser1989/p/18817862)  
· [面试官：如果某个业务量突然提升100倍QPS你会怎么做？](https://www.cnblogs.com/liyongqiang-cc/p/18816921)  
· [聊聊智商税：AI知识库](https://www.cnblogs.com/yexiaochai/p/18817609)  
· [.NET 平台上的开源模型训练与推理进展](https://www.cnblogs.com/whuanle/p/18817790)  
· [Google发布A2A开源协议:“MCP+A2A”成未来标配？](https://www.cnblogs.com/HaiJun-Aion/p/18818310)  

<table><tbody><tr><td colspan="7"><table><tbody><tr><td><a href="https://www.cnblogs.com/youzhibing/p/">&lt;</a></td><td align="center">2025年4月</td><td align="right"><a href="https://www.cnblogs.com/youzhibing/p/">&gt;</a></td></tr></tbody></table></td></tr><tr><th align="center">日</th><th align="center">一</th><th align="center">二</th><th align="center">三</th><th align="center">四</th><th align="center">五</th><th align="center">六</th></tr><tr><td align="center">30</td><td align="center">31</td><td align="center">1</td><td align="center">2</td><td align="center">3</td><td align="center">4</td><td align="center">5</td></tr><tr><td align="center">6</td><td align="center">7</td><td align="center">8</td><td align="center">9</td><td align="center">10</td><td align="center">11</td><td align="center">12</td></tr><tr><td align="center">13</td><td align="center">14</td><td align="center">15</td><td align="center">16</td><td align="center">17</td><td align="center">18</td><td align="center">19</td></tr><tr><td align="center">20</td><td align="center">21</td><td align="center">22</td><td align="center">23</td><td align="center">24</td><td align="center">25</td><td align="center">26</td></tr><tr><td align="center">27</td><td align="center">28</td><td align="center">29</td><td align="center">30</td><td align="center">1</td><td align="center">2</td><td align="center">3</td></tr><tr><td align="center">4</td><td align="center">5</td><td align="center">6</td><td align="center">7</td><td align="center">8</td><td align="center">9</td><td align="center">10</td></tr></tbody></table>

点击右上角即可分享 ![微信分享提示](https://img2023.cnblogs.com/blog/35695/202309/35695-20230906145857937-1471873834.gif)