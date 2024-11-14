---
title: Logback配置
tags:
  - fleet-note
  - development-framework/logback/configuration
date: 2024-10-29
time: 19:39
aliases:
---
# Demo

```xml
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml" />

    <!--定义参数,后面可以通过${APP_NAME}使用-->
    <property name="APP_NAME" value="start" />
    <property name="LOG_PATH" value="${user.home}/${APP_NAME}/logs" />
    <property name="LOG_FILE" value="${LOG_PATH}/application.log" />

    <appender name="APPLICATION"
        class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--定义日志输出的路径-->
        <file>${LOG_FILE}</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${CONSOLE_LOG_PATTERN}</pattern>
            <charset>utf8</charset>
        </encoder>
    </appender>

    <!--rootLogger是默认的logger-->
    <root level="INFO">
        <!--定义了两个appender，日志会通过往这两个appender里面写-->
        <appender-ref ref="CONSOLE" />
        <appender-ref ref="APPLICATION" />
    </root>

    <!--应用日志-->
    <!--这个logger没有指定appender，它会继承root节点中定义的那些appender-->
    <logger name="com.erpang" level="DEBUG"/>

    <!--数据库日志-->
    <!--由于这个logger自动继承了root的appender，root中已经有stdout的appender了，自己这边又引入了stdout的appender-->
    <!--如果没有设置 additivity="false" ,就会导致一条日志在控制台输出两次的情况-->
    <!--additivity表示要不要使用rootLogger配置的appender进行输出-->
    <logger name="com.apache.ibatis" level="TRACE" additivity="false">
        <appender-ref ref="CONSOLE"/>
    </logger>
</configuration>

```

生产中，可以单独将错误日志储存下来：

```xml
    <appender name="APPLICATION_ERROR"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--Log file path-->
        <file>${ERROR_LOG_FILE}</file>
        <encoder>
            <pattern>${FILE_LOG_PATTERN}</pattern>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${ERROR_LOG_FILE}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>7</maxHistory>
            <maxFileSize>50MB</maxFileSize>
            <totalSizeCap>20GB</totalSizeCap>
        </rollingPolicy>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
```
# References

* [logback-spring.xml配置文件标签（超详解）1、SpringBoot日志框架 市面上的日志框架； JUL、 - 掘金](https://juejin.cn/post/7200549600590282789)*
* [README | logback](https://logbackcn.gitbook.io/logback)