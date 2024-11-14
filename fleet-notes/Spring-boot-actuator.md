---
title: Spring-boot-actuator
tags:
  - fleet-note
  - development-framework/spring-boot/actuator
date: 2024-10-30
time: 09:55
aliases:
---

# 默认开放端点

```json
{
  "_links": {
    "self": {
      "href": "http://localhost:8084/actuator",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8084/actuator/health",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8084/actuator/health/{*path}",
      "templated": true
    }
  }
}
```

# 配置

```yaml
# Spring boot actuator  
management:  
  endpoints:  
    enabled-by-default: on  
    web:  
      base-path: /actuator  
      exposure:  
        include: '*'  
  endpoint:  
    health:  
      show-components: always  
      show-details: always  
  health:  
    defaults:  
      enabled: on
```

开启所有端点：

```json
{
  "_links": {
    "self": {
      "href": "http://localhost:8084/actuator",
      "templated": false
    },
    "beans": {
      "href": "http://localhost:8084/actuator/beans",
      "templated": false
    },
    "caches-cache": {
      "href": "http://localhost:8084/actuator/caches/{cache}",
      "templated": true
    },
    "caches": {
      "href": "http://localhost:8084/actuator/caches",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8084/actuator/health",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8084/actuator/health/{*path}",
      "templated": true
    },
    "info": {
      "href": "http://localhost:8084/actuator/info",
      "templated": false
    },
    "conditions": {
      "href": "http://localhost:8084/actuator/conditions",
      "templated": false
    },
    "shutdown": {
      "href": "http://localhost:8084/actuator/shutdown",
      "templated": false
    },
    "configprops": {
      "href": "http://localhost:8084/actuator/configprops",
      "templated": false
    },
    "configprops-prefix": {
      "href": "http://localhost:8084/actuator/configprops/{prefix}",
      "templated": true
    },
    "env": {
      "href": "http://localhost:8084/actuator/env",
      "templated": false
    },
    "env-toMatch": {
      "href": "http://localhost:8084/actuator/env/{toMatch}",
      "templated": true
    },
    "loggers": {
      "href": "http://localhost:8084/actuator/loggers",
      "templated": false
    },
    "loggers-name": {
      "href": "http://localhost:8084/actuator/loggers/{name}",
      "templated": true
    },
    "heapdump": {
      "href": "http://localhost:8084/actuator/heapdump",
      "templated": false
    },
    "threaddump": {
      "href": "http://localhost:8084/actuator/threaddump",
      "templated": false
    },
    "metrics-requiredMetricName": {
      "href": "http://localhost:8084/actuator/metrics/{requiredMetricName}",
      "templated": true
    },
    "metrics": {
      "href": "http://localhost:8084/actuator/metrics",
      "templated": false
    },
    "scheduledtasks": {
      "href": "http://localhost:8084/actuator/scheduledtasks",
      "templated": false
    },
    "mappings": {
      "href": "http://localhost:8084/actuator/mappings",
      "templated": false
    }
  }
```

# References

* [Spring Boot Actuator 配置和应用 - 蓝迷梦 - 博客园](https://www.cnblogs.com/hewei-blogs/articles/17268678.html)*