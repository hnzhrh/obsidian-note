---
title: MySQL 标准字段
tags:
  - middleware/database/mysql
  - fleet-note
date: 2025-02-17
time: 22:42
aliases:
---

```sql
CREATE TABLE IF NOT EXISTS rta_ad_strategy
(
    id           INT AUTO_INCREMENT COMMENT '广告策略ID'
        PRIMARY KEY,
    name VARCHAR(63) NOT NULL COMMENT '广告策略名称',
    gmt_create   DATETIME  DEFAULT CURRENT_TIMESTAMP,
    gmt_modified DATETIME  DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    creator      VARCHAR(63)                          NOT NULL,
    modifier     VARCHAR(63)                          NOT NULL,
    deleted   TINYINT(1) DEFAULT 0                 NOT NULL
);
```



# Reference