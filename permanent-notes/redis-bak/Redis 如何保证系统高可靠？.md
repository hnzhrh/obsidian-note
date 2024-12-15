---
title: Redis 如何保证系统高可靠？
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2024-12-09
time: 11:48
aliases:
---
# 主从集群

Redis 提供了主从模式，采用读写分离的方式：
* 读操作：主从都接受
* 写操作：主库执行，从库同步

给主库配置从库可以通过三种方式：
* 命令执行：`replicaof masterip masterport`
* 配置文件配置
* 启动参数 `--replicaof`

使用 INFO 查看 REPLICATION 模块信息：
```shell
# Replication
role:master
connected_slaves:1
slave0:ip=172.25.0.101,port=6379,state=online,offset=702,lag=1
master_failover_state:no-failover
master_replid:dee3ffedb04c9e0d1e8afc3b2b3ca3ef1f837e7d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:702
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:702

# Replication
role:slave
master_host:172.25.0.100
master_port:6379
master_link_status:up
master_last_io_seconds_ago:1
master_sync_in_progress:0
slave_read_repl_offset:716
slave_repl_offset:716
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:dee3ffedb04c9e0d1e8afc3b2b3ca3ef1f837e7d
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:716
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:716
```

同步过程主要两个阶段，一个是全量同步，发生在从服务器与主服务器第一次建立连接开始同步的情况。
* 从服务器执行 `replicaof` 
* 从服务器发送 `psync ? -1` 命令
* 主服务器发送 `FULLRESYNC {runId} {offset}` 命令，从服务保存主节点信息
* 主服务器生成 RDB，发送给从服务器，有两种方式
	* 直接生成 RDB，发送给子服务器
	* 无盘复制，不生成 RDB 文件，直接发送给子服务器
* 从服务器清空当前数据，加载 RDB
* 主服务器将新的写命令写入 `repl_backlog_buffer` 缓冲区，从服务器根据自己的 offset 加载对应的增量数据

如果从服务器断开连接过久，可能导致从服务器的offset 已不在主服务器的缓冲区中了，此时会触发全量同步。

一般而言，我们可以调整 repl_backlog_size 这个参数。这个参数和所需的缓冲空间大小有关。缓冲空间的计算公式是：缓冲空间大小 = 主库写入命令速度 * 操作大小 - 主从库间网络传输命令速度 * 操作大小。在实际应用中，考虑到可能存在一些突发的请求压力，我们通常需要把这个缓冲空间扩大一倍，即 repl_backlog_size = 缓冲空间大小 * 2，这也就是 repl_backlog_size 的最终值。

举个例子，如果主库每秒写入 2000 个操作，每个操作的大小为 2KB，网络每秒能传输 1000 个操作，那么，有 1000 个操作需要缓冲起来，这就至少需要 2MB 的缓冲空间。否则，新写的命令就会覆盖掉旧操作了。为了应对可能的突发压力，我们最终把 repl_backlog_size 设为 4MB。这样一来，增量复制时主从库的数据不一致风险就降低了。不过，如果并发请求量非常大，连两倍的缓冲空间都存不下新操作请求的话，此时，主从库数据仍然可能不一致。

> Redis和客户端通信也好，和从库通信也好，Redis都需要给分配一个内存buffer进行数据交互，客户端是一个client，从库也是一个client，我们每个client连上Redis后，Redis都会分配一个client buffer，所有数据交互都是通过这个buffer进行的：Redis先把数据写到这个buffer中，然后再把buffer中的数据发到client socket中再通过网络发送出去，这样就完成了数据交互。所以主从在增量同步时，从库作为一个client，也会分配一个buffer，只不过这个buffer专门用来传播用户的写命令到从库，保证主从数据一致，我们通常把它叫做replication buffer。
> 再延伸一下，既然有这个内存buffer存在，那么这个buffer有没有限制呢？如果主从在传播命令时，因为某些原因从库处理得非常慢，那么主库上的这个buffer就会持续增长，消耗大量的内存资源，甚至OOM。所以Redis提供了client-output-buffer-limit参数限制这个buffer的大小，如果超过限制，主库会强制断开这个client的连接，也就是说从库处理慢导致主库内存buffer的积压达到限制后，主库会强制断开从库的连接，此时主从复制会中断，中断后如果从库再次发起复制请求，那么此时可能会导致恶性循环，引发复制风暴，这种情况需要格外注意。

# 哨兵机制

当使用主从机制时，如果主库挂了，则需要人为的干预，因此Redis 引入了哨兵机制解决这个问题。

哨兵有三个作用：
* 监控主库和从库，判断是否上下线
* 选择出新主库
* 通知从库和客户端
	* 通知从库与新的主库数据同步
	* 通知客户端与新的主库进行连接

哨兵使用PING 命令检测网络连接状态，用来判断实例的状态，判断分为主观下线和客观下线，当多台哨兵判定为主观下线且大于用户配置的quorum 参数，设定为客观下线。

## 哨兵之间如何通信？

配置哨兵时需要执行主库：
```c
sentinel monitor <master-name> <ip> <redis-port> <quorum> 
```
哨兵集群通过在主库上 pub/sub 做到互相之间通信。
哨兵直接如何建立联系：
哨兵都订阅 `__sentinel__:hello` 频道，各自发布自己的 ip 和端口，相互之间就可以得知地址进行通信了。

## 哨兵如何得知从库？

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

## 如何选定新主库？

如何选定主库：
1. 筛选从库
	1. 筛选出在线的从库
	2. 判断之前的网络连接状态，筛选出稳定的（配置：`down-after-milliseconds`）
2. 打分
	1. 优先级高的从库得高分（配置：`slave-priority`）
	2. 和旧主库同步程度最接近得从库得高分 (`slave_repl_offset:86363`)
	3. RunId 越小的得分越高


在选主时，除了要检查从库的当前在线状态，还要判断它之前的网络连接状态。如果从库总是和主库断连，而且断连次数超出了一定的阈值，我们就有理由相信，这个从库的网络状况并不是太好，就可以把这个从库筛掉了。

具体怎么判断呢？你使用配置项 down-after-milliseconds * 10。其中，down-after-milliseconds 是我们认定主从库断连的最大连接超时时间。如果在 down-after-milliseconds 毫秒内，主从节点都没有通过网络联系上，我们就可以认为主从节点断连了。如果发生断连的次数超过了 10 次，就说明这个从库的网络状况不好，不适合作为新主库。

## 如何通知从库和客户端的新主库地址？

应用程序不感知服务的中断，还需要哨兵和客户端做些什么？当哨兵完成主从切换后，客户端需要及时感知到主库发生了变更，然后把缓存的写请求写入到新库中，保证后续写请求不会再受到影响，具体做法如下：

哨兵提升一个从库为新主库后，哨兵会把新主库的地址写入自己实例的pubsub（switch-master）中。客户端需要订阅这个pubsub，当这个pubsub有数据时，客户端就能感知到主库发生变更，同时可以拿到最新的主库地址，然后把写请求写到这个新主库即可，这种机制属于哨兵主动通知客户端。

如果客户端因为某些原因错过了哨兵的通知，或者哨兵通知后客户端处理失败了，安全起见，客户端也需要支持主动去获取最新主从的地址进行访问。

所以，客户端需要访问主从库时，不能直接写死主从库的地址了，而是需要从哨兵集群中获取最新的地址（sentinel get-master-addr-by-name命令），这样当实例异常时，哨兵切换后或者客户端断开重连，都可以从哨兵集群中拿到最新的实例地址。

一般Redis的SDK都提供了通过哨兵拿到实例地址，再访问实例的方式，我们直接使用即可，不需要自己实现这些逻辑。当然，对于只有主从实例的情况，客户端需要和哨兵配合使用，而在分片集群模式下，这些逻辑都可以做在proxy层，这样客户端也不需要关心这些逻辑了，Codis就是这么做的。

## 哪个哨兵执行主从切换？

任何一个实例只要自身判断主库“主观下线”后，就会给其他实例发送 is-master-down-by-addr 命令。接着，其他实例会根据自己和主库的连接情况，做出 Y 或 N 的响应，Y 相当于赞成票，N 相当于反对票。

一个哨兵获得了仲裁所需的赞成票数后，就可以标记主库为“客观下线”。这个所需的赞成票数是通过哨兵配置文件中的 quorum 配置项设定的。例如，现在有 5 个哨兵，quorum 配置的是 3，那么，一个哨兵需要 3 张赞成票，就可以标记主库为“客观下线”了。这 3 张赞成票包括哨兵自己的一张赞成票和另外两个哨兵的赞成票。

此时，这个哨兵就可以再给其他哨兵发送命令，表明希望由自己来执行主从切换，并让所有其他哨兵进行投票。这个投票过程称为“Leader 选举”。因为最终执行主从切换的哨兵称为 Leader，投票过程就是确定 Leader。

在投票过程中，任何一个想成为 Leader 的哨兵，要满足两个条件：第一，拿到半数以上的赞成票；第二，拿到的票数同时还需要大于等于哨兵配置文件中的 quorum 值。以 3 个哨兵为例，假设此时的 quorum 设置为 2，那么，任何一个想成为 Leader 的哨兵只要拿到 2 张赞成票，就可以了。


## 日志

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


# Reference
* [06 \| 数据同步：主从库如何实现数据一致？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/272852)
* [08 \| 哨兵集群：哨兵挂了，主从库还能切换吗？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/275337)
* [Redis replication \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/replication/)
* [High availability with Redis Sentinel \| Docs](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
* [redis/sentinel.conf at 7.0.0 · redis/redis · GitHub](https://github.com/redis/redis/blob/7.0.0/sentinel.conf)
