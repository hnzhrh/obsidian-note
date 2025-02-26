---
title: ebay-pipeline-scheduler 设计
tags:
  - fleet-note
  - project/ebay/pipeline-schedule
date: 2025-02-17
time: 13:48
aliases:
---
# 任务状态机设计






# 定时任务设计

多 Scheduler 多 Worker 设计

任务状态设计：
* PENDING
* ASSIGNED
* RUNNING
* SUCCEED
* FAILED
* CANCELED




# SQL

```sql

create schema auto_etl_for_analyst collate utf8mb4_general_ci;

```


# Reference
