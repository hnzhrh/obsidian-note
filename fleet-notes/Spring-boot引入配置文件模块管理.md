---
title: Spring-boot引入配置文件模块管理
tags:
  - fleet-note
  - development-framework/spring-boot/configuration
date: 2024-10-29
time: 19:17
aliases:
---
# 多Profile配置


```yaml
spring:  
  profiles:  
    active: local
```

可以创建 `application-local.yaml` 的配置文件，此时引入的为 `local` 环境配置

# 引入其他配置

在配置文件较多且配置项也很多的情况下，可以通过引入文件进行分模块配置。

例如创建`datasource-local.yaml` 用来管理本地的数据库配置：
```yaml
spring:  
  datasource:  
    username: root  
    password: 123456  
    url: jdbc:mysql://${MYSQL_HOST}:3306/cola_dev?serverTimezone=UTC&useUnicode=true&characterEncoding=utf-8  
    driver-class-name: com.mysql.cj.jdbc.Driver
```

在 `application-local.yaml` 中引入：

```yaml
spring:  
  config:  
	import: classpath:datasource-${spring.profiles.active}.yaml
```

使用占位符引入对应环境的配置文件