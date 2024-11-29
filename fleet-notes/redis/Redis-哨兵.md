---
title: Redis-哨兵
tags:
  - fleet-note
date: 2024-11-28
time: 13:52
aliases:
---
哨兵主要做三件事：
1. 监控主库、从库（ping pang）
2. 选主
3. 通知客户端和从库

哨兵是存在误判的情况下，比如因为网络拥堵、Redis 实例本身压力较大的情况下可能无法响应哨兵的 Ping 请求。

通常引入哨兵集群进行判断，如果是从库，则直接判定为主观下线。如果是主库，当有 N 个哨兵实例时，如果有 N/2+1 个实例认为主观下线，则判定为主库客观下线。

如何选定主库：
1. 筛选从库
	1. 筛选出在线的从库
	2. 判断之前的网络连接状态，筛选出稳定的（配置：`down-after-milliseconds`）
2. 打分
	1. 优先级高的从库得高分（配置：`slave-priority`）
	2. 和旧主库同步程度最接近得从库得高分 (`slave_repl_offset:86363`)
	3. RunId 越小的得分越高



哨兵集群

配置哨兵时需要执行主库：
```c
sentinel monitor <master-name> <ip> <redis-port> <quorum> 
```
哨兵集群通过在主库上 pub/sub 做到互相之间通信。
哨兵直接如何建立联系：
哨兵都订阅 `__sentinel__:hello` 频道，各自发布自己的 ip 和端口，相互之间就可以得知地址进行通信了。

哨兵如何得知从库：
哨兵通过调用主库的 info 命令可以直接获取到从库的地址
```c
# Replication
role:master
connected_slaves:2
slave0:ip=192.168.208.3,port=6379,state=online,offset=200124,lag=1
slave1:ip=192.168.208.2,port=6379,state=online,offset=200124,lag=1
master_failover_state:no-failover
master_replid:64caf7d9a47231bc5839bb9c776114ba2b3a49fd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:200262
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:200262
```


这里有个问题，假设有主机：
```c
redis-1 
redis-2 
redis-3

sentinel-1
sentinel-2
sentinel-3
```
配置 redis 主库为 1 ，当 1 下线后，哨兵切换到主为 2
如果全部都重启了，就会出问题，因为 sentinel 已经修改了 monitor 的主为 2，而重启之后 redis 集群的主其实是 1.
# 从库的数据同步进度

在Redis中，`slave_repl_offset` 和 `slave_read_repl_offset` 是与复制（replication）相关的两个概念。它们用于跟踪主从复制过程中数据同步的状态。下面是对这两个配置项的详细说明：

### slave_repl_offset

- **定义**：`slave_repl_offset` 不是一个配置参数，而是一个信息字段，它表示从节点（slave/replica）已经成功处理的数据流字节数。这个值可以用来确定从节点相对于主节点的数据同步进度。
- **作用**：这个字段对于监控和管理Redis复制过程非常有用。通过比较主节点的`master_repl_offset`与从节点的`slave_repl_offset`，你可以了解复制滞后的情况，即从节点落后于主节点多少数据。
- **获取方式**：可以通过`INFO replication`命令来查看当前从节点的`slave_repl_offset`值。

### slave_read_repl_offset

- **定义**：`slave_read_repl_offset` 也不是一个配置参数，而是一个内部状态变量，它反映了从节点读取到的数据流字节数，但可能还没有被处理或应用。
- **作用**：这个值有助于理解从节点接收数据的速度与其处理数据速度之间的关系。如果`slave_read_repl_offset`显著高于`slave_repl_offset`，这可能意味着从节点正在努力追赶以处理接收到的数据。
- **获取方式**：同样地，这个值也可以通过`INFO replication`命令查看。

### 总结

- **`slave_repl_offset`** 指示了从节点已经完全处理并持久化的数据量。
- **`slave_read_repl_offset`** 则指示了从节点已经从网络上接收到但尚未处理的数据量。

这两个偏移量对于诊断复制延迟、评估复制性能以及确保数据一致性非常重要。在实际操作中，管理员通常会关注这些值的变化，以确保复制进程正常运行，并及时发现潜在的问题。例如，如果发现 `slave_repl_offset` 长时间没有变化，可能是由于网络问题或者从节点的处理能力不足导致的。


# 日志

创建三台 sentinel，三台 redis 实例，一主两从。

```log
第一台sentinel

1:X 29 Nov 2024 03:11:59.835 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:X 29 Nov 2024 03:11:59.835 # Redis version=7.0.15, bits=64, commit=00000000, modified=0, pid=1, just started
1:X 29 Nov 2024 03:11:59.835 # Configuration loaded
1:X 29 Nov 2024 03:11:59.836 * monotonic clock: POSIX clock_gettime
1:X 29 Nov 2024 03:11:59.836 * Running mode=sentinel, port=26379.
1:X 29 Nov 2024 03:11:59.837 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:X 29 Nov 2024 03:11:59.837 # Sentinel ID is cb7dd39708f7f735254b2faf738dfe4417f28adc
1:X 29 Nov 2024 03:11:59.837 # +monitor master mymaster 172.25.0.100 6379 quorum 2
1:X 29 Nov 2024 03:11:59.839 * +slave slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 03:11:59.845 * Sentinel new configuration saved on disk
1:X 29 Nov 2024 03:11:59.845 * +slave slave 172.25.0.101:6379 172.25.0.101 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 03:11:59.850 * Sentinel new configuration saved on disk

第二胎sentinel
1:X 29 Nov 2024 03:11:59.783 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:X 29 Nov 2024 03:11:59.783 # Redis version=7.0.15, bits=64, commit=00000000, modified=0, pid=1, just started
1:X 29 Nov 2024 03:11:59.783 # Configuration loaded
1:X 29 Nov 2024 03:11:59.783 * monotonic clock: POSIX clock_gettime
1:X 29 Nov 2024 03:11:59.785 * Running mode=sentinel, port=26379.
1:X 29 Nov 2024 03:11:59.785 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:X 29 Nov 2024 03:11:59.785 # Sentinel ID is cb7dd39708f7f735254b2faf738dfe4417f28adc
1:X 29 Nov 2024 03:11:59.785 # +monitor master mymaster 172.25.0.100 6379 quorum 2
1:X 29 Nov 2024 03:11:59.786 * +slave slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 03:11:59.795 * Sentinel new configuration saved on disk
1:X 29 Nov 2024 03:11:59.795 * +slave slave 172.25.0.101:6379 172.25.0.101 	

第三台sentinel
1:X 29 Nov 2024 03:11:59.877 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:X 29 Nov 2024 03:11:59.877 # Redis version=7.0.15, bits=64, commit=00000000, modified=0, pid=1, just started
1:X 29 Nov 2024 03:11:59.877 # Configuration loaded
1:X 29 Nov 2024 03:11:59.878 * monotonic clock: POSIX clock_gettime
1:X 29 Nov 2024 03:11:59.880 * Running mode=sentinel, port=26379.
1:X 29 Nov 2024 03:11:59.880 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:X 29 Nov 2024 03:11:59.880 # Sentinel ID is cb7dd39708f7f735254b2faf738dfe4417f28adc
1:X 29 Nov 2024 03:11:59.880 # +monitor master mymaster 172.25.0.100 6379 quorum 2
1:X 29 Nov 2024 03:11:59.881 * +slave slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 03:11:59.889 * Sentinel new configuration saved on disk
1:X 29 Nov 2024 03:11:59.889 * +slave slave 172.25.0.101:6379 172.25.0.101 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 03:11:59.895 * Sentinel new configuration saved on disk
```


主服务器
```c
1:C 29 Nov 2024 03:11:59.316 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 29 Nov 2024 03:11:59.316 # Redis version=7.0.15, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 29 Nov 2024 03:11:59.316 # Configuration loaded
1:M 29 Nov 2024 03:11:59.317 * monotonic clock: POSIX clock_gettime
1:M 29 Nov 2024 03:11:59.318 * Running mode=standalone, port=6379.
1:M 29 Nov 2024 03:11:59.318 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 29 Nov 2024 03:11:59.318 # Server initialized
1:M 29 Nov 2024 03:11:59.318 # WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can can also cause failures without low memory condition, see https://github.com/jemalloc/jemalloc/issues/1328. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 29 Nov 2024 03:11:59.326 * Creating AOF base file appendonly.aof.1.base.rdb on server start
1:M 29 Nov 2024 03:11:59.335 * Creating AOF incr file appendonly.aof.1.incr.aof on server start
1:M 29 Nov 2024 03:11:59.335 * Ready to accept connections
1:M 29 Nov 2024 03:11:59.365 * Replica 172.25.0.102:6379 asks for synchronization
1:M 29 Nov 2024 03:11:59.365 * Full resync requested by replica 172.25.0.102:6379
1:M 29 Nov 2024 03:11:59.365 * Replication backlog created, my new replication IDs are '36cd39bbe4611f7bc3fd4e1a60cb3d8866373c0e' and '0000000000000000000000000000000000000000'
1:M 29 Nov 2024 03:11:59.365 * Delay next BGSAVE for diskless SYNC
1:M 29 Nov 2024 03:11:59.421 * Replica 172.25.0.101:6379 asks for synchronization
1:M 29 Nov 2024 03:11:59.421 * Full resync requested by replica 172.25.0.101:6379
1:M 29 Nov 2024 03:11:59.421 * Delay next BGSAVE for diskless SYNC
1:M 29 Nov 2024 03:12:04.353 * Starting BGSAVE for SYNC with target: replicas sockets
1:M 29 Nov 2024 03:12:04.354 * Background RDB transfer started by pid 18
18:C 29 Nov 2024 03:12:04.354 * Fork CoW for RDB: current 0 MB, peak 0 MB, average 0 MB
1:M 29 Nov 2024 03:12:04.355 # Diskless rdb transfer, done reading from pipe, 2 replicas still up.
1:M 29 Nov 2024 03:12:04.364 * Background RDB transfer terminated with success
1:M 29 Nov 2024 03:12:04.364 * Streamed RDB transfer with replica 172.25.0.102:6379 succeeded (socket). Waiting for REPLCONF ACK from slave to enable streaming
1:M 29 Nov 2024 03:12:04.364 * Synchronization with replica 172.25.0.102:6379 succeeded
1:M 29 Nov 2024 03:12:04.364 * Streamed RDB transfer with replica 172.25.0.101:6379 succeeded (socket). Waiting for REPLCONF ACK from slave to enable streaming
1:M 29 Nov 2024 03:12:04.364 * Synchronization with replica 172.25.0.101:6379 succeeded
```
从服务器：
```c
1:C 29 Nov 2024 03:11:59.405 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 29 Nov 2024 03:11:59.405 # Redis version=7.0.15, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 29 Nov 2024 03:11:59.405 # Configuration loaded
1:S 29 Nov 2024 03:11:59.405 * monotonic clock: POSIX clock_gettime
1:S 29 Nov 2024 03:11:59.406 * Running mode=standalone, port=6379.
1:S 29 Nov 2024 03:11:59.406 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:S 29 Nov 2024 03:11:59.406 # Server initialized
1:S 29 Nov 2024 03:11:59.406 # WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can can also cause failures without low memory condition, see https://github.com/jemalloc/jemalloc/issues/1328. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:S 29 Nov 2024 03:11:59.416 * Creating AOF base file appendonly.aof.1.base.rdb on server start
1:S 29 Nov 2024 03:11:59.420 * Creating AOF incr file appendonly.aof.1.incr.aof on server start
1:S 29 Nov 2024 03:11:59.420 * Ready to accept connections
1:S 29 Nov 2024 03:11:59.420 * Connecting to MASTER 172.25.0.100:6379
1:S 29 Nov 2024 03:11:59.420 * MASTER <-> REPLICA sync started
1:S 29 Nov 2024 03:11:59.420 * Non blocking connect for SYNC fired the event.
1:S 29 Nov 2024 03:11:59.420 * Master replied to PING, replication can continue...
1:S 29 Nov 2024 03:11:59.421 * Partial resynchronization not possible (no cached master)
1:S 29 Nov 2024 03:12:04.354 * Full resync from master: 36cd39bbe4611f7bc3fd4e1a60cb3d8866373c0e:857
1:S 29 Nov 2024 03:12:04.355 * MASTER <-> REPLICA sync: receiving streamed RDB from master with EOF to disk
1:S 29 Nov 2024 03:12:04.355 * MASTER <-> REPLICA sync: Flushing old data
1:S 29 Nov 2024 03:12:04.355 * MASTER <-> REPLICA sync: Loading DB in memory
1:S 29 Nov 2024 03:12:04.363 * Loading RDB produced by version 7.0.15
1:S 29 Nov 2024 03:12:04.363 * RDB age 0 seconds
1:S 29 Nov 2024 03:12:04.363 * RDB memory usage when created 1.14 Mb
1:S 29 Nov 2024 03:12:04.363 * Done loading RDB, keys loaded: 0, keys expired: 0.
1:S 29 Nov 2024 03:12:04.363 * MASTER <-> REPLICA sync: Finished with success
1:S 29 Nov 2024 03:12:04.363 * Creating AOF incr file temp-appendonly.aof.incr on background rewrite
1:S 29 Nov 2024 03:12:04.363 * Background append only file rewriting started by pid 20
20:C 29 Nov 2024 03:12:04.369 * Successfully created the temporary AOF base file temp-rewriteaof-bg-20.aof
20:C 29 Nov 2024 03:12:04.370 * Fork CoW for AOF rewrite: current 0 MB, peak 0 MB, average 0 MB
1:S 29 Nov 2024 03:12:04.439 * Background AOF rewrite terminated with success
1:S 29 Nov 2024 03:12:04.439 * Successfully renamed the temporary AOF base file temp-rewriteaof-bg-20.aof into appendonly.aof.2.base.rdb
1:S 29 Nov 2024 03:12:04.439 * Successfully renamed the temporary AOF incr file temp-appendonly.aof.incr into appendonly.aof.2.incr.aof
1:S 29 Nov 2024 03:12:04.443 * Removing the history file appendonly.aof.1.incr.aof in the background
1:S 29 Nov 2024 03:12:04.443 * Removing the history file appendonly.aof.1.base.rdb in the background
1:S 29 Nov 2024 03:12:04.449 * Background AOF rewrite finished successfully
```

下线主服务器, 这一台依然是 slave
```c
1:M 29 Nov 2024 05:35:18.832 # Connection with master lost.
1:M 29 Nov 2024 05:35:18.832 * Caching the disconnected master state.
1:S 29 Nov 2024 05:35:18.832 * Connecting to MASTER 172.25.0.101:6379
1:S 29 Nov 2024 05:35:18.833 * MASTER <-> REPLICA sync started
1:S 29 Nov 2024 05:35:18.833 * REPLICAOF 172.25.0.101:6379 enabled (user request from 'id=7 addr=172.25.0.200:44974 laddr=172.25.0.102:6379 fd=14 name=sentinel-8ee76e02-cmd age=74 idle=0 flags=x db=0 sub=0 psub=0 ssub=0 multi=4 qbuf=339 qbuf-free=20135 argv-mem=4 multi-mem=180 rbs=4096 rbp=2048 obl=45 oll=0 omem=0 tot-mem=25648 events=r cmd=exec user=default redir=-1 resp=2')
1:S 29 Nov 2024 05:35:18.842 # CONFIG REWRITE executed with success.
1:S 29 Nov 2024 05:35:18.842 * Non blocking connect for SYNC fired the event.
1:S 29 Nov 2024 05:35:18.842 * Master replied to PING, replication can continue...
1:S 29 Nov 2024 05:35:18.842 * Trying a partial resynchronization (request 7df8044b71398e4ac3f38fc64afc30937ff58e8c:8443).
1:S 29 Nov 2024 05:35:18.843 * Successful partial resynchronization with master.
1:S 29 Nov 2024 05:35:18.843 # Master replication ID changed to 8cbe44fd13e701294f18f0818bb2b96f7e4bf235
1:S 29 Nov 2024 05:35:18.843 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.```

Sentinel 日志：
```c
1:X 29 Nov 2024 05:35:17.586 # +sdown master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:17.653 # +odown master mymaster 172.25.0.100 6379 #quorum 2/2
1:X 29 Nov 2024 05:35:17.653 # +new-epoch 1
1:X 29 Nov 2024 05:35:17.653 # +try-failover master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:17.662 * Sentinel new configuration saved on disk
1:X 29 Nov 2024 05:35:17.662 # +vote-for-leader 8ee76e02f3e0d5cd2eda5d79d548e2373e6d1a96 1
1:X 29 Nov 2024 05:35:17.675 # d81d09c2ea46e0f7a2cc0b73e509e9952aafb967 voted for 8ee76e02f3e0d5cd2eda5d79d548e2373e6d1a96 1
1:X 29 Nov 2024 05:35:17.675 # 0d977d90cc41118c1997026e280b9b5e917be7da voted for 8ee76e02f3e0d5cd2eda5d79d548e2373e6d1a96 1
1:X 29 Nov 2024 05:35:17.728 # +elected-leader master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:17.728 # +failover-state-select-slave master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:17.819 # +selected-slave slave 172.25.0.101:6379 172.25.0.101 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:17.819 * +failover-state-send-slaveof-noone slave 172.25.0.101:6379 172.25.0.101 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:17.919 * +failover-state-wait-promotion slave 172.25.0.101:6379 172.25.0.101 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:18.738 * Sentinel new configuration saved on disk
1:X 29 Nov 2024 05:35:18.738 # +promoted-slave slave 172.25.0.101:6379 172.25.0.101 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:18.738 # +failover-state-reconf-slaves master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:18.832 * +slave-reconf-sent slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:19.763 * +slave-reconf-inprog slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:19.764 * +slave-reconf-done slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:19.816 # -odown master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:19.816 # +failover-end master mymaster 172.25.0.100 6379
1:X 29 Nov 2024 05:35:19.816 # +switch-master mymaster 172.25.0.100 6379 172.25.0.101 6379
1:X 29 Nov 2024 05:35:19.816 * +slave slave 172.25.0.102:6379 172.25.0.102 6379 @ mymaster 172.25.0.101 6379
1:X 29 Nov 2024 05:35:19.816 * +slave slave 172.25.0.100:6379 172.25.0.100 6379 @ mymaster 172.25.0.101 6379
1:X 29 Nov 2024 05:35:19.826 * Sentinel new configuration saved on disk
1:X 29 Nov 2024 05:35:49.830 # +sdown slave 172.25.0.100:6379 172.25.0.100 6379 @ mymaster 172.25.0.101 6379
1:X 29 Nov 2024 05:36:04.407 # -sdown slave 172.25.0.100:6379 172.25.0.100 6379 @ mymaster 172.25.0.101 6379
1:X 29 Nov 2024 05:36:14.425 * +convert-to-slave slave 172.25.0.100:6379 172.25.0.100 6379 @ mymaster 172.25.0.101 6379
1:X 29 Nov 2024 05:37:02.559 # +sdown slave 172.25.0.100:6379 172.25.0.100 6379 @ mymaster 172.25.0.101 6379
```

```shell
./redis-6379/redis-log ./redis-6380/redis-log ./sentinel-26379/sentinel-log ./sentinel-26380/sentinel-log ./sentinel-26381/sentinel-log


```
# References
* [07 | 哨兵机制：主库挂了，如何不间断服务？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/274483)
* [High availability with Redis Sentinel | Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)