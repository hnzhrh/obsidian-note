---
title: "轻量级分布式任务调度平台(二、XXL-JOB环境搭建集成springboot)"
tags:
  - "clippings literature-note"
date: 2025-02-26
time: 2025-02-26T17:34:05+08:00
source: "https://www.cnblogs.com/MrYuChen-Blog/p/14804036.html"
---
> 接上文......

## (7) XXL-JOB环境搭建#
### (7.1) 源码结构#

通过上面给出的源码下载地址，我们将源码clone到IDEA中，如下：  
[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163710433-824439078.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163710433-824439078.jpg)

### (7.2) 初始化数据库#

初始化脚本在上面源码目录的 `/doc/db/tables_xxl_job.sql` ，将此脚本在`MySQL`数据库中执行一遍。  
执行完毕，会在MySQL数据库中生成如下 8 张表：  
[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163709905-1175003939.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163709905-1175003939.jpg)

### (7.3) 配置调度中心#

调度中心就是源码中的 `xxl-job-admin` 工程，我们需要将其配置成自己需要的调度中心，通过该工程我们能够以图形化的方式统一管理任务调度平台上调度任务，负责触发调度执行。

#### (7.3.1) 修改调度中心配置文件

```java
### web
server.port=8080
server.servlet.context-path=/xxl-job-admin

### actuator
management.server.servlet.context-path=/actuator
management.health.mail.enabled=false

### resources
spring.mvc.servlet.load-on-startup=0
spring.mvc.static-path-pattern=/static/**
spring.resources.static-locations=classpath:/static/

### freemarker
spring.freemarker.templateLoaderPath=classpath:/templates/
spring.freemarker.suffix=.ftl
spring.freemarker.charset=UTF-8
spring.freemarker.request-context-attribute=request
spring.freemarker.settings.number_format=0.##########

### mybatis
mybatis.mapper-locations=classpath:/mybatis-mapper/*Mapper.xml
#mybatis.type-aliases-package=com.xxl.job.admin.core.model

### xxl-job, datasource
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=root_pwd
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

### datasource-pool
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.maximum-pool-size=30
spring.datasource.hikari.auto-commit=true
spring.datasource.hikari.idle-timeout=30000
spring.datasource.hikari.pool-name=HikariCP
spring.datasource.hikari.max-lifetime=900000
spring.datasource.hikari.connection-timeout=10000
spring.datasource.hikari.connection-test-query=SELECT 1
spring.datasource.hikari.validation-timeout=1000

### xxl-job, email
spring.mail.host=smtp.qq.com
spring.mail.port=25
spring.mail.username=xxx@qq.com
spring.mail.from=xxx@qq.com
spring.mail.password=xxx
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true
spring.mail.properties.mail.smtp.socketFactory.class=javax.net.ssl.SSLSocketFactory

### xxl-job, access token
xxl.job.accessToken=

### xxl-job, i18n (default is zh_CN, and you can choose "zh_CN", "zh_TC" and "en")
xxl.job.i18n=zh_CN

## xxl-job, triggerpool max size
xxl.job.triggerpool.fast.max=200
xxl.job.triggerpool.slow.max=100

### xxl-job, log retention days
xxl.job.logretentiondays=30
```

**注意**：基本上上面的配置文件我们需要修改的只有第 5 点，**修改数据库的地址**，这要与我们前面初始化的数据库名称径，用户名密码保持一致；第二个就是修改第 6 点，**报警邮箱**，因为该工程任务失败后有失败告警功能，可以通过邮件来提醒，如果我们需要此功能，可以配置一下。

#### (7.3.2) 部署调度中心

[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163709491-1167822789.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163709491-1167822789.jpg)

#### (7.3.3)访问调度中心管理界面

地址：

> [http://localhost:8080/xxl-job-admin/](http://localhost:8080/xxl-job-admin/)  
> [![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163708894-1438975177.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163708894-1438975177.jpg)

### (7.4) 创建执行器项目#

下面我以创建一个 `springboot` 版本的执行器为例来介绍:

在源码中作者已经贴心的给出了多种执行器项目示例，可根据你的喜好直接将其部署作为你自己的执行器  
现以集成到现有项目为例，将执行器集成到现有的一个Spring-Boot项目Athena中去  
**步骤一：在你的项目里引入xxl-job-core的依赖**

```java
<!-- xxl-rpc-core -->
<dependency>
	<groupId>com.xuxueli</groupId>
	<artifactId>xxl-job-core</artifactId>
	<version>2.3.0</version>
</dependency>
```

**步骤二：执行器配置**  
在创建好的`springboot` 项目的配置文件 `application.yml`添加如下配置：

```java
#项目端口号
server.port=8081
#日志文件
logging.config= classpath:logback.xml
#调度中心部署跟地址：如调度中心集群部署存在多个地址则用逗号分隔。
#执行器将会使用该地址进行"执行器心跳注册"和"任务结果回调"。
xxl.job.admin.addresses =http://127.0.0.1:8080/xxl-job-admin

### 执行器通讯TOKEN [选填]：非空时启用；
xxl.job.accessToken =

#分别配置执行器的名称、ip地址、端口号
#注意：如果配置多个执行器时，防止端口冲突
xxl.job.executor.appname =executorDemo
xxl.job.executor.address =
xxl.job.executor.ip = 127.0.0.1
xxl.job.executor.port = 9999

#执行器运行日志文件存储的磁盘位置，需要对该路径拥有读写权限
xxl.job.executor.logpath= D:/data/xxl-job/jobhandler
#执行器Log文件定期清理功能，指定日志保存天数，日志文件过期自动删除。限制至少保持3天，否则功能不生效；
#-1表示永不删除
xxl.job.executor.logretentiondays= -1
```

> **这里需要注意的是：配置执行器的名称、IP地址、端口号，后面如果配置多个执行器时，要防止端口冲突。再就是执行器的名称，我们后面会到上一步的调度中心管理界面进行对应配置。**

**步骤三：载入配置文件**  
在项目中创建 XxlJobConfig.class 文件：

```java
package com.example.studyprojects.config;

import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class XxlJobConfig {
    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.appname}")
    private String appname;

    @Value("${xxl.job.executor.address}")
    private String address;

    @Value("${xxl.job.executor.ip}")
    private String ip;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;

    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(appname);
        xxlJobSpringExecutor.setAddress(address);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);

        return xxlJobSpringExecutor;
    }
}
```

**XXL-JOB执行器的相关配置项:**

- `xxl.job.admin.addresses`

> 调度中心的部署地址。若调度中心采用集群部署，存在多个地址，则用逗号分隔。执行器将会使用该地址进行”执行器心跳注册”和”任务结果回调”。

- `xxl.job.executor.appname`

> 执行器的应用名称，它是执行器心跳注册的分组依据。

- `xxl.job.executor.ip`

> 执行器的IP地址，用于”调度中心请求并触发任务”和”执行器注册”。执行器IP默认为空，表示自动获取IP。多网卡时可手动设置指定IP，手动设置IP时将会绑定Host。

- `xxl.job.executor.port`

> 执行器的端口号，默认值为9999。单机部署多个执行器时，注意要配置不同的执行器端口。

- `xxl.job.accessToken`

> 执行器的通信令牌，非空时启用。

- `xxl.job.executor.logpath`

> 执行器输出的日志文件的存储路径，需要拥有该路径的读写权限。

- xxl.job.executor.logretentiondays

> **执行器日志文件的定期清理功能**，指定日志保存天数，日志文件过期自动删除。限制至少保存3天，否则功能不生效。

**步骤四：创建任务**

　 　在项目中创建一个Handler，用于执行我们想要执行的东西，这里我只是简单的打印一行日志：  
　>　XXL-JOB, Hello World!!!

```java
/*
 任务示例
 */
@Component
public class JobHandlerDemo {

    @XxlJob(value = "demoJobHandler")
    public ReturnT<String> execute(String s) throws Exception {
        System.out.println("=====hello world=====");
//        XxlJobLogger.log("XXL-JOB, Hello World.");
        return ReturnT.SUCCESS;
    }
}
```
### (8) 调度中心中配置执行器，添加任务#

> 调度中心前面我们已经配置好了，启动该配置中心，进入http://localhost:8080/xxl-job-admin 界面。

#### (8.1) 配置执行器

点击 执行器管理----》新增执行器---》，如下如下界面，然后填充此表格，点击保存即可。

[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163708290-981300137.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163708290-981300137.jpg)

**相关参数说明：**

- **AppName**：是每个执行器集群的唯一标识`AppName`, 执行器会周期性以`AppName`为对象进行`自动注册`。可通过该配置自动发现注册成功的执行器, 供任务调度时使用;
- **名称**：执行器的名称, 因为AppName限制字母数字等组成,可读性不强, 名称为了提高执行器的可读性;
- **注册方式**：调度中心获取执行器地址的方式，

> 1. 自动注册：执行器自动进行执行器注册，调度中心通过底层注册表可以动态发现执行器机器地址；
> 2. 手动录入：人工手动录入执行器的地址信息，多地址逗号分隔，供调度中心使用；

- **机器地址**："注册方式"为"手动录入"时有效，支持人工维护执行器的地址信息；

#### (8.2) 添加新任务

点击 任务管理---》新增任务---》

[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163707852-504001434.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163707852-504001434.jpg)

**相关参数说明：**

- **执行器**：任务的绑定的执行器，任务触发调度时将会自动发现注册成功的执行器, 实现任务自动发现功能; 另一方面也可以方便的进行任务分组。每个任务必须绑定一个执行器, 可在 "执行器管理" 进行设置。
- **任务描述**：任务的描述信息，便于任务管理；
- **路由策略**：当执行器集群部署时，提供丰富的路由策略，包括；  
FIRST（第一个）：固定选择第一个机器；  
LAST（最后一个）：固定选择最后一个机器；  
ROUND（轮询）：  
RANDOM（随机）：随机选择在线的机器；  
CONSISTENT\_HASH（一致性HASH）：每个任务按照Hash算法固定选择某一台机器，且所有任务均匀散列在不同机器上。  
LEAST\_FREQUENTLY\_USED（最不经常使用）：使用频率最低的机器优先被选举；  
LEAST\_RECENTLY\_USED（最近最久未使用）：最久为使用的机器优先被选举；  
FAILOVER（故障转移）：按照顺序依次进行心跳检测，第一个心跳检测成功的机器选定为目标执行器并发起调度；  
BUSYOVER（忙碌转移）：按照顺序依次进行空闲检测，第一个空闲检测成功的机器选定为目标执行器并发起调度；  
SHARDING\_BROADCAST(分片广播)：广播触发对应集群中所有机器执行一次任务，同时系统自动传递分片参数；可根据分片参数开发分片任务；
- **Cron**：触发任务执行的Cron表达式；
- **运行模式**：  
​ BEAN模式：任务以JobHandler方式维护在执行器端；需要结合 "JobHandler" 属性匹配执行器中任务；  
　　GLUE模式(Java)：任务以源码方式维护在调度中心；该模式的任务实际上是一段继承自IJobHandler的Java类代码并 "groovy" 源码方式维护，它在执行器项目中运行，可使用@Resource/@Autowire注入执行器里中的其他服务；  
　　GLUE模式(Shell)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "shell" 脚本；  
　　GLUE模式(Python)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "python" 脚本；  
　　GLUE模式(PHP)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "php" 脚本；  
　　GLUE模式(NodeJS)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "nodejs" 脚本；  
　　GLUE模式(PowerShell)：任务以源码方式维护在调度中心；该模式的任务实际上是一段 "PowerShell" 脚本；
- **JobHandler**：运行模式为 "BEAN模式" 时生效，对应执行器中新开发的JobHandler类“@JobHandler”注解自定义的value值；
- **阻塞处理策略**：调度过于密集执行器来不及处理时的处理策略；  
单机串行（默认）：调度请求进入单机执行器后，调度请求进入FIFO队列并以串行方式运行；  
丢弃后续调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，本次请求将会被丢弃并标记为失败；  
覆盖之前调度：调度请求进入单机执行器后，发现执行器存在运行的调度任务，将会终止运行中的调度任务并清空队列，然后运行本地调度任务；
- **子任务**：每个任务都拥有一个唯一的任务ID(任务ID可以从任务列表获取)，当本任务执行结束并且执行成功时，将会触发子任务ID所对应的任务的一次主动调度。
- **任务超时时间**：支持自定义任务超时时间，任务运行超时将会主动中断任务；
- **失败重试次数**；支持自定义任务失败重试次数，当任务失败时将会按照预设的失败重试次数主动进行重试；
- **报警邮件**：任务调度失败时邮件通知的邮箱地址，支持配置多邮箱地址，配置多个邮箱地址时用逗号分隔；
- **负责人**：任务的负责人；
- **执行参数**：任务执行所需的参数，多个参数时用逗号分隔，任务执行时将会把多个参数转换成数组传入；

### (9) 启动项目 测试任务#

配置完执行器以及任务，我们只需要启动该任务，便可以运行了。

[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163707297-399809891.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163707297-399809891.jpg)

启动之后，我们查看后台日志：

[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163706910-2003140402.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163706910-2003140402.jpg)  
定时任务触发成功！！

调度中心查询日志：  
[![](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163706178-1125471506.jpg)](https://img2020.cnblogs.com/blog/2026387/202105/2026387-20210521163706178-1125471506.jpg)  
调度成功！！  
执行成功！！

---

## 轻量级分布式任务调度平台XXL-JOB系列#

> [轻量级分布式任务调度平台(一、 XXL-JOB介绍、原理、工作流程)](https://www.cnblogs.com/MrYuChen-Blog/p/14804019.html)  
> [轻量级分布式任务调度平台(二、XXL-JOB环境搭建集成springboot)](https://www.cnblogs.com/MrYuChen-Blog/p/14804036.html)