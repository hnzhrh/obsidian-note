---
title: Redis部署
tags:
  - fleet-note
  - middleware/redis/ops
date: 2024-11-28
time: 16:21
aliases:
---


# Sentinel

如果使用域名，需要修改 sentinel 配置为：
```c
# HOSTNAMES SUPPORT
#
# Normally Sentinel uses only IP addresses and requires SENTINEL MONITOR
# to specify an IP address. Also, it requires the Redis replica-announce-ip
# keyword to specify only IP addresses.
#
# You may enable hostnames support by enabling resolve-hostnames. Note
# that you must make sure your DNS is configured properly and that DNS
# resolution does not introduce very long delays.
#
SENTINEL resolve-hostnames yes

# When resolve-hostnames is enabled, Sentinel still uses IP addresses
# when exposing instances to users, configuration files, etc. If you want
# to retain the hostnames when announced, enable announce-hostnames below.
#
SENTINEL announce-hostnames yes
```


SENTINEL 选举需要配置
```
bind 0.0.0.0
```


```shell
tail -f ./redis-6379/redis-log > ./logs-monitor/redis-6379-redis-log &
tail -f ./redis-6380/redis-log > ./logs-monitor/redis-6380-redis-log &
tail -f ./redis-6381/redis-log > ./logs-monitor/redis-6381-redis-log &

tail -f ./sentinel-26379/sentinel-log > ./logs-monitor/sentinel-26379-sentinel-log &
tail -f ./sentinel-26380/sentinel-log > ./logs-monitor/sentinel-26379-sentinel-log &
tail -f ./sentinel-26381/sentinel-log > ./logs-monitor/sentinel-26379-sentinel-log &
wait
```
# References