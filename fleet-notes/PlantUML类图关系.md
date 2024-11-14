---
title: PlantUML类图关系表示方法
tags:
  - fleet-note
  - design/uml
date: 2024-10-29
time: 13:46
aliases:
---
在 UML 中，有多种关系可以用来描述类、接口、包等之间的关系。以下是常见的关系及其对应的 PlantUML 表示法：

1.  **实现（Realization）**：
   - 表示一个类实现一个接口。
   - 写法：`ClassA ..|> InterfaceB`

2. **依赖（Dependency）**：
   - 表示一个类依赖于另一个类，通常是通过参数、返回值等形式。
   - 写法：`ClassA --> ClassB`

3. **关联（Association）**：
   - 表示两个类之间的静态关系，可以是单向或双向的。
   - 写法：`ClassA -- ClassB`（双向关联）
   - 单向关联可表示为：`ClassA --> ClassB`

4. **聚合（Aggregation）**：
   - 表示一个类是另一个类的部分，但可以独立存在。
   - 写法：`ClassA o-- ClassB`

5. **组合（Composition）**：
   - 表示一个类是另一个类的部分，并且其生命周期依赖于包含它的类。
   - 写法：`ClassA *-- ClassB`

6. **依赖注入（Dependency Injection）**：
   - 表示一个类通过构造函数或方法接收另一个类的实例。
   - 写法：`ClassA --> ClassB : injects`

7. **扩展（Extend）**：
   - 表示一个用例在另一个用例的基础上扩展功能。
   - 写法：`UseCaseA --|> UseCaseB : extends`

# References
* [DesignPattern/design-pattern/谈一谈自己对依赖、关联、聚合和组合之间区别的理解.md at master · JackChan1999/DesignPattern · GitHub](https://github.com/JackChan1999/DesignPattern/blob/master/design-pattern/%E8%B0%88%E4%B8%80%E8%B0%88%E8%87%AA%E5%B7%B1%E5%AF%B9%E4%BE%9D%E8%B5%96%E3%80%81%E5%85%B3%E8%81%94%E3%80%81%E8%81%9A%E5%90%88%E5%92%8C%E7%BB%84%E5%90%88%E4%B9%8B%E9%97%B4%E5%8C%BA%E5%88%AB%E7%9A%84%E7%90%86%E8%A7%A3.md)*