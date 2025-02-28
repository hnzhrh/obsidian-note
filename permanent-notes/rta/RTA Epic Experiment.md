---
Title: RTA Epic Experiment
tags:
  - permanent-note
  - project/rta
  - product/user-story
Date: 2024-11-14
Time: 14:16
aliases:
---
# 创建实验策略

![](https://img.shields.io/badge/priority-high-blue)

**As** 广告平台的用户
**I want** 创建实验策略
**So that** 可以配置人群过滤、频次控制、动态出价等配置项，达到精准投放，提升投放效率，拿到更优质的流量

## Acceptance criteria

* **Given** 我已登录广告平台，进入实验策略管理页面，看到已创建的实验策略列表，且有新建策略按钮，也可编辑已有策略
* **When** 点击新建策略按钮
* **Then** 进入新建实验策略页面

* **Given** 我已在新建实验策略页面
* **When** 尝试输入一个已存在的实验策略名称
* **Then** 系统提示实验策略已存在

* **Given** 我已在新建实验策略页面，并完成以下实验策略配置
* **When** 点击创建
* **Then** 系统创建实验策略且返回实验策略管理页面
## Relation

* [Prototype](https://www.processon.com/embed/67335167c3e0723987eabe64?cid=67335168c3e0723987eabe67)

# 配置实验策略-人群过滤

![](https://img.shields.io/badge/priority-high-blue)

**As** 广告平台的用户
**I want** 配置需要定向的人群包，需要排除的人群包
**So that** 达到精准定向、过滤黑名单的目的

## Acceptance criteria

* **Given** 我已在新建实验策略页面，默认开启了人群过滤开关
* **When** 在人群包列表中选择一个人群包，点击定向
* **Then** 已定向人群包列表中展示该人群包
* **When** 在人群包列表中选择一个人群包，点击排除
* **Then** 已排除人群包列表中展示该人群包
## Relation

* [Prototype](https://www.processon.com/embed/67335167c3e0723987eabe64?cid=67335168c3e0723987eabe67)


# 配置实验策略-频次控制

![](https://img.shields.io/badge/priority-high-blue)

**As** 广告平台的用户
**I want** 根据点击、展示配置用户的广告频次
**So that** 过滤点击欺诈和低质量流量

## Acceptance criteria

* **Given** 我已在新建实验策略页面
* **When** 关闭频次控制
* **Then** 不展示频次控制配置项

* **Given** 我已在新建实验策略页面，
* **When** 开启频次控制
* **Then** 展示频次控制配置项

* **Given** 我已开启频次控制
* **When** 选择点击或者展示频控标的
* **Then** 输入次数上线和追溯时长

* **Given** 我已选择频控标的
* **When** 尝试输入次数上限 > 99
* **Then** 系统提示错误

* **Given** 我已选择频控标的
* **When** 尝试输入追溯市场 > 15
* **Then** 系统提示错误
## Relation

* [Prototype](https://www.processon.com/embed/67335167c3e0723987eabe64?cid=67335168c3e0723987eabe67)


# 配置实验策略-动态出价

![](https://img.shields.io/badge/priority-high-blue)

**As** 广告平台的用户
**I want** 开启动态出价
**So that** 可以借助算法模型能力，针对不同广告，优选高质量用户，提高出价，提升投放效率
## Relation

* [Prototype](https://www.processon.com/embed/67335167c3e0723987eabe64?cid=67335168c3e0723987eabe67)

# 创建、配置并开启实验

![](https://img.shields.io/badge/priority-high-blue)

**As** 广告平台的用户
**I want** 创建实验，可以配置对照组和实验组，各组还能配置流量占比，选择实验策略和广告策略
**So that** 对流量进行 AB 实验，通过后链路数据分析筛选效果更加的投放方案

## Acceptance criteria

* **Given** 我已登录到广告平台，进入 AB 实验管理页面，看到已创建的 AB 实验列表，且有新建实验按钮
* **When** 点击新建实验按钮
* **Then** 弹出新建实验对话框

* **Given** 我已在新建实验对话框中
* **When** 尝试输入已有实验名称
* **Then** 系统提示实验已存在

* **Given** 我已完成媒体类型选择、实验名称输入、实验目标选择
* **When** 点击提交
* **Then** 系统创建实验且跳转到实验配置页面

* **Given** 我已进入实验配置页面
* **When** 点击新增实验组
* **Then** 界面出现一个新的实验组，默认流量为 0%，

* **Given** 我已完成实验分组
* **When** 点击某个组的推全按钮
* **Then** 系统进行推全按钮，且更改状态为推全中

* **Given** 我已完成实验配置，且返回实验列表
* **When** 点击开启实验，该实验的实验组配置已推全
* **Then** 开启实验
* **When** 点击开启实验，该实验的实验组配置未推全
* **Then** 系统提示实验配置未推全
## Relation

* [实验创建Prototype](https://www.processon.com/embed/67345634be9d2e22e51342af?cid=67345634be9d2e22e51342b27)
* [实验组配置Prototype](https://www.processon.com/embed/67345b25b61b2f442a8ccb53?cid=67345b25b61b2f442a8ccb56)
