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

# Reference
* [06 \| 数据同步：主从库如何实现数据一致？-Redis核心技术与实战-极客时间](https://time.geekbang.org/column/article/272852)