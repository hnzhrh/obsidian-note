---
Title: RTA-epic-advertise-strategy
tags:
  - product/user-story
  - permanent-note
  - project/rta
Date: 2024-11-14
Time: 15:22
aliases: 
epic:
---
# 创建广告策略

![](https://img.shields.io/badge/priority-high-blue)

**As** RTA 广告平台的用户
**I want** 创建一个新的广告策略，这个策略可以选择媒体类型，也可以添加广告账号和广告单元
**So that** 统一管理广告账号和广告单元，使用更方便，回传 RTA 请求更简洁，降低传输带宽消耗

## Acceptance criteria

* **Given** 我已登录到广告平台，进入广告策略管理页面，看到已创建的广告策略列表，且有新建策略按钮，也可编辑已有策略
* **When** 点击了新建策略按钮
* **Then** 进入新建策略页面

* **Given** 我已进入新建策略页面，看到了策略名称输入框，可以选择媒体类型的单选框，且看到三个选择区，分别是广告账号，广告单元和已添加
* **When** 我尝试输入一个已存在的策略名称
* **Then** 系统提示策略名称已存在

* **Given** 我已在新建策略页面
* **When** 选择媒体类型
* **Then** 广告账号选择区显示该媒体类型下的已有广告账号
* **When** 点击广告账号的选择框
* **Then** 已添加列表展示该广告账号但不展示该账号的广告单元
* **When** 点击广告账号
* **Then** 广告单元列表展示该广告账号的广告单元，且可多选
* **When** 点击广告单元的选择框
* **Then** 已添加列表展示该广告单元和广告账号

* **Given** 我已在新建策略页面，且已完成策略名称的输入、媒体类型的选择、广告账号和单元的添加
* **When** 点击提交按钮
* **Then** 系统保存新广告策略且返回广告策略管理页面

## Relation

* [Prototype](https://www.processon.com/embed/67332df46b5a4a4addbd8fd1?cid=67332df46b5a4a4addbd8fd4)

# 编辑广告策略

![](https://img.shields.io/badge/priority-high-blue)

**As** RTA 广告平台的用户
**I want** 编辑一个新的广告策略，可以添加、删除广告账号和单元
**So that** 更改广告策略的设置

## Acceptance criteria

* **Given** 我已登录到广告平台，进入广告策略管理页面，看到已创建的广告策略列表
* **When** 点击一个现有广告策略的编辑按钮
* **Then** 重定向到该广告策略的编辑页面

* **Given** 我已在广告策略编辑页面
* **When** 尝试更改策略名称和媒体类型
* **Then** 置灰无法选择

* **Given** 我已在广告策略编辑页面，已选择列表显示已绑定的广告账号和广告单元
* **When** 在已选择列表取消勾选某个广告账号或者广告单元，添加广告账号或广告单元，并点击提交按钮
* **Then** 弹出对话框提示，显示新增加和已解绑的广告账号和广告单元
* **When** 点击确认
* **Then** 系统保存修改且返回到广告策略管理页面
## Relation

* [Prototype](https://www.processon.com/embed/6735b7476b5a4a4addc13a4e?cid=6735b7476b5a4a4addc13a514)