---
title: Redis记录
tags:
  - fleet-note
date: 2024-11-22
time: 18:18
aliases:
---
# 测试一下写入

10 个线程写入 100 W
```java
@PostMapping  
public Response add() {  
    int numOfThread = 10;  
    CountDownLatch countDownLatch = new CountDownLatch(numOfThread);  
    for (int i = 0; i < numOfThread; i++) {  
        new Thread(() -> {  
            try {  
                for (int j = 0; j < 100000; j++) {  
                    DevicePO devicePO = new DevicePO();  
                    devicePO.setType("Android");  
                    devicePO.setIncludeAudience(randomAudience());  
                    devicePO.setExcludeAudience(randomAudience());  
                    deviceCache.putDevice(randomDeviceId(), devicePO, randomBidInfoMap());  
                }  
            } finally {  
                countDownLatch.countDown();  
            }  
        }).start();  
    }  
    try {  
        countDownLatch.await();  
    } catch (InterruptedException e) {  
        throw new RuntimeException(e);  
    }  
    return Response.buildSuccess();  
}
```

Redis 配置：

```yaml
  # Redis config
  data:
    redis:
      database: 0
      host: ${REDIS_HOST:localhost}
      port: 6379
      #      password: 123456
      timeout: 100ms
      connectTimeout: 5s
      clientName: ${spring.application.name}
      lettuce:
        pool:
          # 连接池最大连接数（使用负值表示没有限制） 默认 8
          max-active: 8
          # 连接池中的最大空闲连接 默认 8
          max-idle: 8
          # 连接池最大阻塞等待时间（使用负值表示没有限制） 默认 -1
          max-wait: -1
          # 连接池中的最小空闲连接 默认 0
          min-idle: 0
```

最终结果：436 s

改大线程数为 50
配置：
```yaml
          # 连接池最大连接数（使用负值表示没有限制） 默认 8
          max-active: 64
          # 连接池中的最大空闲连接 默认 8
          max-idle: 32
```

100 多秒搞定
# 内存占用

在使用 Protobuf 压缩后，从 2.3 G 下降到 900 多 M，效果相当明显

```shell
# Memory
used_memory:965854776
used_memory_human:921.11M
used_memory_rss:991653888
used_memory_rss_human:945.71M
used_memory_peak:966884144
used_memory_peak_human:922.09M
used_memory_peak_perc:99.89%
used_memory_overhead:49305104
used_memory_startup:862080
used_memory_dataset:916549672
used_memory_dataset_perc:94.98%
allocator_allocated:966443480
allocator_active:967131136
allocator_resident:986247168
total_system_memory:16637091840
total_system_memory_human:15.49G
used_memory_lua:34816
used_memory_vm_eval:34816
used_memory_lua_human:34.00K
used_memory_scripts_eval:232
number_of_cached_scripts:1
```

修改参数：

```redis
hash-max-listpack-value 64
```

```shell
# Memory
used_memory:729855480
used_memory_human:696.04M
used_memory_rss:758456320
used_memory_rss_human:723.32M
used_memory_peak:966884144
used_memory_peak_human:922.09M
used_memory_peak_perc:75.49%
used_memory_overhead:49325576
used_memory_startup:862080
used_memory_dataset:680529904
used_memory_dataset_perc:93.35%
```

效果还是相当明显的

# References