---
title: RTA-event-storming
tags:
  - fleet-note
  - project/rta
  - product/skills/event-storming
date: 2024-11-13
time: 20:01
aliases:
---
# Ubiquitous language

| Bounded Context | 中文名         | 英文名                                   | 解释                                               |
| --------------- | ----------- | ------------------------------------- | ------------------------------------------------ |
|                 | RTA 广告策略    | RTA Ad Strategy                       | 用于回复给媒体的 RTA 广告策略，绑定多个广告媒体账号或者广告单元               |
|                 | RTA 实验策略    | RTA Experiment Strategy               | RTA 实验策略，包括多项实验策略配置项，比如开启人群包（定向和排除）、开启频控、开启动态出价等 |
|                 | RTA 实验策略配置项 | RTA Experiment Strategy Configuration | RTA 实验策略的配置项                                     |
|                 | RTA ID      | RTA ID                                | 用于快手媒体的线下约定的特有 ID，用于 RTA广告策略的创建和绑定               |
|                 | 人群包         | Custome Audience                      | 包含设备号用于定向或者排除                                    |
|                 | 频次控制        | Frequency Limit                       | 可以控制广告用户点击、展示在设定窗口大小下的最大点击、观看次数                  |

# Event Storming

[RTA-event-storming.excelidraw](RTA-event-storming.excelidraw.md)
# Domain Design
## Context Map

```plantuml
@startuml
package "Rta" {
	[AdStrategy]
	[ExperimentStrategy]
	[Experiment]
}
@enduml
``` 
## UML

```plantuml
@startuml
package "AdStrategy Aggregate" {
	class "AdStrategy" {
		-id:Long
		-name:String
		-media:MediaTypeEnum
		-adAccounts:List<AdAccount>
	}
	
	enum MediaTypeEnum {
		TENCENT
		OCEAN_ENGINE
		KUAI_SHOU
	}
	
	class "AdAccount" {
		-id:Long
		-adGroups:AdGroup
	}
	
	class "AdGroup" {
		-id:Long
	}
	
	class "AdStrategyDomainService" {
		+void bind(AdStrategy adStrategy, AdAccount adAccount)
		+void unbind(AdStrategy adStrategy, AdAccount adAccount)
	}

	AdStrategy o-- MediaTypeEnum
	AdStrategy "1" o-- "0..*" AdAccount
	AdAccount "1" o-- "0..*" AdGroup
}


package "ExperimentStrategy Aggregate" {
	class "ExperimentStrategy" {
		-id:Long
		-name:String
		-customAudience:CustomAudience
		-frequencyLimit:FrequencyLimit
		-dynamicBid:DynamicBid
	}
	
	class "CustomAudience" {
		-switchOn:bool
		-includeIds:List<String>
		-excludeIds:List<String>
		+boolean isMatchInclude(List<String> includeIds)
		+boolean isMatchExclude(List<String> excludeIds)
	}
	
	class "FrequencyLimit" {
		-switchOn:bool
		-target:FrenquencyLimitTargetEnum
		-period:Short
		-maxLimit:Short
		+boolean isWithMaxLimit(Long lastCountTimestamp, List<Short> counts)
	}
	
	enum "FrenquencyLimitTargetEnum" {
		CLICK
		SHOW
	}
	
	class "DynamicBid" {
		-switchOn:bool
		-bidRatio:Float
	}

	
	ExperimentStrategy o-- CustomAudience
	ExperimentStrategy "1" o-- "0..1" FrequencyLimit
	ExperimentStrategy "1" o-- "0..1" DynamicBid
	ExperimentStrategy o-- FrenquencyLimitTargetEnum
}

package "Experiment Aggregate" {
	class "Experiment" {
		-id:Integer
		-name:String
		-marketingGoal:MarketingGoalEnum
		-controlGroup:ControlGroup
		-experimentGroups:List<ExperimentGroup>
	}
	
	enum "MarketingGoalEnum" {
		CUSTOM_ACQUISTION:拉新
		CUSTOM_REACTIVATION:拉活
	}
	
	class "ControlGroup" {
		-buckets:List<Short>
		-adStrategyIds:List<Long>
		-experimentStrategyId:Long
	}
	
	class "ExperimentGroup" {
		-buckets:List<Short>
		-adStrategyIds:List<Long>
		-experimentStrategyId:Long
	}
	
	Experiment o-- MarketingGoalEnum
	Experiment o-- ControlGroup
	Experiment "1" o-- "0..*" ExperimentGroup
}
@enduml
```





# API Design


# Detail Design

## UML

## 

## Sequence




# References