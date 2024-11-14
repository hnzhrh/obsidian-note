---
title: RTA-brain-storming
tags:
  - fleet-note
  - project/rta
date: 2024-11-12
time: 08:52
aliases:
---
# 参与方

* 媒体，流量提供者，严格限制，15 W QPS、RT < 60 ms
* 广告主
* RTA application

# 名词解释


# User story drafts


| As  | I want      | So that                             |
| --- | ----------- | ----------------------------------- |
| 广告主 | 登录          | 使用系统                                |
| 广告主 | 创建 RTA 广告策略 | 减少服务器压力，方便统计和管理                     |
| 广告主 | 广告策略绑定      | 策略维度管理广告账号和广告单元，减少 RTA 应用和媒体双方服务器压力 |
| 广告主 | 创建实验        | 针对 RTA 流量分组实验，结合后链路数据分析，提升广告效益      |
| 广告主 | 创建实验策略      | 做到频控配置，配置定向人群包和排除人群包，绑定 RTA 实验      |
| 广告主 | 配置实验        | 配置实验分组占比，绑定实验策略                     |
|     |             |                                     |

# 设计思路

设备号添加人群包字段
频控的维度怎么设计？

# References

* [RTA策略id及策略管理工具说明文档 - 轻雀文档](https://docs.qingque.cn/d/home/eZQAw8bRE1EuBLIco0auVg7kl?identityId=1oEDL7Yu55g#section=h.9xbonows3edc)