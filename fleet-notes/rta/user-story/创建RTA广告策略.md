---
Title: 创建RTA广告策略
tags:
  - product/user-story
  - project/rta
Date: 2024-11-12
Time: 10:37
aliases: 
epic:
---
![](https://img.shields.io/badge/priority-high-blue)
## 创建RTA广告策略

**As** 广告主
**I want** 创建 RTA 广告策略
**So that** 通过 RTA 广告策略管理广告账号和广告单元，回复 RTA 请求时直接回复策略，降低响应大小，减少带宽占用，降低双方（RTA 应用和媒体侧）服务器压力

## Acceptance criteria

* **Given** 描述了测试或用户故事的前提条件，即在执行操作之前系统应该处于的状态或已知的事实
* **When** 描述了用户执行的操作或触发的事件
* **Then** 描述了预期的结果或系统应该做出的响应
---
* **Given** 用户已经与媒体运营同步开白，且创建了开发者账户（advertiser_id），申请开通了 RTA 通道（rta_id）
* **When** 用户通过Sidecar 进入
* **Then** 内容区显示当前已创建的策略
---
* **Given** 内容区已显示当前已创建策略
* **When** 用户点击创建新策略
* **Then** 展示 RTA 广告创建页面
	* 输入 RTA 广告策略名称
	* 选择媒体名称
	* 绑定广告账号和广告单元
---
* **Given** 用户已进入 RTA 广告创建页面
* **When** 用户选择广告媒体
* **Then** 广告账号与单元内容第一列显示当前媒体的广告账号，滚动分页
---
* **Given** 用户已进入 RTA 广告创建页面，
* **When** 用户勾选广告账号
* **Then** 添加到已添加列中
---
* **Given** 用户已进入 RTA 广告创建页面，
* **When** 用户勾选某个媒体账号，并点击展开按钮
* **Then** 第二列展示选中媒体账号的广告单元
---
* **Given** 用户已进入 RTA 广告创建页面，且已展开了广告媒体账号，中间列显示了该账号的广告单元
* **When** 用户勾选广告单元
* **Then** 添加到已添加列中
---
* **Given** 用户已选择了广告账号与单元
* **When** 点击提交按钮
* **Then** 执行创建请求，返回已创建的广告策略界面
## Relation

* [Prototype](https://www.processon.com/embed/67332df46b5a4a4addbd8fd1?cid=67332df46b5a4a4addbd8fd4)