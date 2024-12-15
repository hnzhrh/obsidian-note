---
title: Spring Boot 能同时处理多少请求？
tags:
  - development-framework/spring-boot
  - fleet-note
date: 2024-12-09
time: 12:41
aliases:
---
`spring-configuration-metadata.json` 配置文件中，默认指定了 Tomcat 最大连接数：
```json
{
  "name": "server.tomcat.max-connections",
  "type": "java.lang.Integer",
  "description": "Maximum number of connections that the server accepts and processes at any given time. Once the limit has been reached, the operating system may still accept connections based on the \"acceptCount\" property.",
  "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat",
  "defaultValue": 8192
}
```

当连接数大于最大连接数时，新的请求将进入队列等待调度，默认配置为：
```json
{
  "name": "server.tomcat.accept-count",
  "type": "java.lang.Integer",
  "description": "Maximum queue length for incoming connection requests when all possible request processing threads are in use.",
  "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat",
  "defaultValue": 100
}
```


# Reference
* [Spring Boot: How Many Requests Can Spring Boot Handle Simultaneously? \| by Oliver Foster \| Medium](https://medium.com/@haiou-a/spring-boot-how-many-requests-can-spring-boot-handle-simultaneously-a57b41bdba6a)