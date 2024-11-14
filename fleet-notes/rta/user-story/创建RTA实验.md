---
Title: 创建RTA实验
tags:
  - product/user-story
Date: 2024-11-13
Time: 14:12
aliases: 
epic:
---
# 创建 RTA 实验

![](https://img.shields.io/badge/priority-high-blue)

**As** 广告主
**I want** 创建 RTA 实验
**So that** 可以对 RTA 实验策略、广告策略进行科学效果评估

## Acceptance criteria

* **Given** 用户通过侧边导航栏进入实验管理页面
* **When** 点击创建实验
* **Then** 弹出创建实验对话框
---
* **Given** 用户可选择媒体，输入实验名称，选择业务目标
* **When** 用户完成输入和选择后点击下一步
* **Then** 进入实验配置步骤

# 配置 RTA 实验

**As** 广告主
**I want** 配置流量分桶，配置实验策略和广告策略
**So that** 通过后链路数据分析投放效果

## Acceptance criteria

* **Given** 进入实验配置页面
* **When** 点击新增实验组
* **Then** 新增一个实验组以供配置
---
* **Given** 已新增一个实验组
* **When** 配置流量比例（最大 90%，10%为对照组），配置实验策略和广告策略
* **Then** 点击推全配置
---
* **Given** 已推全
* **When** 算法中心初始化数据后回调通知数据已初始化后
* **Then** 广告主可以开启实验
# Relation

* [配置实验 prototype](https://www.processon.com/embed/67345b25b61b2f442a8ccb53?cid=67345b25b61b2f442a8ccb56)
* [新建实验](https://www.processon.com/embed/67345634be9d2e22e51342af?cid=67345634be9d2e22e51342b2)