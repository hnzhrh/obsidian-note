---
title: DDD
tags:
  - fleet-note
  - architecture/ddd
date: 2024-11-06
time: 18:34
aliases:
---

# 目标

* Distil knowledge
* Ubiquitous Language
* Shared Understanding

# 两个阶段

* [DDD战略设计](DDD战略设计.md)
* [DDD战术设计](DDD战术设计.md)

领域驱动设计（Domain-Driven Design, DDD）是一种软件开发方法论，它强调业务领域的重要性，鼓励开发者与业务专家合作，以便更好地理解和实现复杂的业务需求。DDD 包含了战略设计和战术设计两大方面，它们各自关注不同的层次和方面，但都是为了确保软件系统能够有效地支持业务目标。

### 战略设计

**战略设计** 主要关注高层次的业务理解，包括领域建模、子域划分、限界上下文的定义以及上下文映射。这一阶段的目标是确保团队对业务有深刻的理解，并能够将这种理解转化为有效的软件架构。以下是战略设计的核心要点：

- **领域模型**：建立领域模型，这是对业务领域的抽象，有助于团队成员之间形成共同的理解。
- **子域划分**：根据业务复杂度和功能的不同，将领域细分为多个子域，如核心域、支撑域和通用域。
- **限界上下文**：明确各个子域的边界，确保每个子域内的术语和概念在该上下文内具有一致性和无歧义性。
- **上下文映射**：定义不同限界上下文之间的关系和交互方式，如共享内核、客户/供应商、防腐层等模式，以减少不同子域间的耦合。

### 战术设计

**战术设计** 则更加关注具体的技术实现细节，包括如何在代码层面体现领域模型，如何实现聚合、实体、值对象、领域服务等。战术设计的目标是确保领域模型能够有效地转换为软件实现，同时保持良好的可维护性和扩展性。以下是战术设计的核心要点：

- **聚合**：将一组密切相关的事物（实体和值对象）组织在一起，形成一个整体，聚合内部的事物对外部来说是不可分割的。
- **聚合根**：每个聚合都有一个聚合根，它是聚合的入口点，外界只能通过聚合根与聚合内部的对象交互。
- **实体**：拥有唯一标识的对象，即使其他属性相同，只要标识不同就被视为不同的实体。
- **值对象**：没有唯一标识的对象，它的值决定了它的等价性。
- **领域服务**：当某个操作不属于任何实体或值对象时，可以将其定义为领域服务，领域服务通常包含跨越多个聚合的业务逻辑。
- **应用服务**：位于应用层，负责协调领域层的操作，处理业务用例的执行顺序及结果的组装，应用层不能直接调用[领域实体](领域实体.md) 的方法，而应该通过[领域服务](领域服务.md) 进行调用（因为[[领域实体]]可能在业务发展过程中可能会被拆分出去作为单独的微服务部署，这时候[领域实体](领域实体.md)的方法实际上是访问不到的，但可以通过 RPC 或者事件调用到[领域服务](领域服务.md)），应用服务也可以直接调用基础层，比如缓存、文件操作等。
- **资源库**：提供了一种访问聚合的方式，隐藏了数据访问的细节，使得领域对象与数据存储机制解耦。
- **领域事件**：表示领域中发生的重要事件，可以用来触发其他业务逻辑或跨限界上下文的通信。
# 方法论

* [事件风暴](事件风暴.md)

# 核心概念

* 领域
	* 子域
		* 核心域
		* 通用域
		* 支撑域

# References

* 【从 MVC 到 DDD 重构，我们有了新想法！—— Java 项目重构经验总结，彻底搞清楚 MVC 和 DDD】 https://www.bilibili.com/video/BV1Dz4y1V7zf/?share_source=copy_web&vd_source=3eb28f54d17403f9d05aaa09bef421a4
* 【1-熊锐-DDD 的理解及实践探讨】 https://www.bilibili.com/video/BV1pG4y167jF/?share_source=copy_web&vd_source=3eb28f54d17403f9d05aaa09bef421a4
* [DDD领域建模实战——四色建模法 - Su的技术博客](https://blog.verysu.com/aritcle/ddd/1823)
* [Domain-Driven Design in ProductLand - Alberto Brandolini - DDD Europe 2022 - YouTube](https://youtu.be/ufdcfe8VmHM?si=DdWgZ656jdZbn3KN)
* [DDD Explained in 9 MINUTES | What is Domain Driven Design? - YouTube](https://youtu.be/kbGYy49fCz4?si=Xg6OW_jV5yMW6Ec8)
* [Domain Driven Design: What You Need To Know - YouTube](https://youtu.be/4rhzdZIDX_k?si=ZiQU2D38xei0KiaN)
* [STOP dogmatic Domain Driven Design - YouTube](https://youtu.be/8XmXhXH_q90?si=YrRqIkQekhJXA0p9)
* [Fetching Title#348o](https://youtu.be/o-ym035R1eY?si=jvgf_LbXThM-_R9k)