---
Title: 创建RTA实验策略
tags:
  - product/user-story
  - project/rta
Date: 2024-11-12
Time: 20:38
aliases: 
epic:
---


![](https://img.shields.io/badge/priority-high-blue)
## 创建 RTA 实验策略

**As** 广告主
**I want** 创建 RTA 实验策略
**So that** 实现频率控制，设置定向和排除人群包，实现精准投放，提升广告投放效率

## Acceptance criteria

* **Given** 用户登录进入系统
* **When** 通过 Sidecar 进入策略管理
* **Then** 展示策略列表
	* 策略 ID
	* 策略名称
	* 业务目标
	* 创建时间
	* 操作
		* 编辑
		* 删除
---
* **Given** 用户已进入策略管理页面
* **When** 点击新建策略
* **Then** 弹出策略新建对话框
---
* **Given** 用户已在新建策略对话框重
* **When** 用户输入策略名称，选择定向和排除人群包，可开启频控并设置频控标的，追溯时长，次数上限后，选择是否开启动态出价，点击创建
* **Then** 创建策略成功则进入策略管理页面
## Relation

* [Prototype](https://www.processon.com/embed/67335167c3e0723987eabe64?cid=67335168c3e0723987eabe67)
# References