---
title: Redis 主从复制配置
tags:
  - permanent-note
  - middleware/redis/reliability
date: 2025-03-12
time: 13:19
aliases: 
done: false
---

# 1 `replica-serve-stale-data yes`

* 设置为 `yes`，副本和主服务器断连时任然可以对外提供服务，保证了可用性，但可能读取到过期数据（第一次同步时，可能读取到空数据），提高了可用性，降低了数据一致性。
* 设置为 `no`，任何访问命令客户端将收到报错 `MASTERDOWN Link with MASTER is down and replica-serve-stale-data is set to 'no'`，提高了数据一致性，降低了可用性。
	* 除了这些命令，`INFO, REPLICAOF, AUTH, SHUTDOWN, REPLCONF, ROLE, CONFIG, SUBSCRIBE,UNSUBSCRIBE, PSUBSCRIBE, PUNSUBSCRIBE, PUBLISH, PUBSUB, COMMAND, POST,HOST，LATENCY`

# 2 `repl-diskless-sync yes`

Redis 同步策略有两种：
* `disk`
* `socket`
全量同步时 RDB 文件传输有两种方式：
* `Disk-backed`，主进程 `fork` 子进程生成 RDB 文件后，主进程依次发送 RDB 给副本，其他副本需要等待当前传输结束
	* 分块发送
	* 非阻塞 IO
	* `sendfile` 零拷贝系统调用
* `Diskless`，主进程 `fork` 子进程直接通过 `socket` 传输给副本。可以通过配置 `repl-diskless-sync-delay 5` 设置 5 s 等待时间，等待更多的副本同步请求后并行传输

# 3 `repl-diskless-sync-max-replicas 0`

`repl-diskless-sync-delay` 等待期间，如果无盘同步请求达到这个配置，则直接进行传输。

# 4 `repl-ping-replica-period 10`

主服务器发送 `PING` 的间隔为 10 s

# 5 `repl-timeout 60`

设置副本超时时间为 60 s，需要比配置 `repl-ping-replica-period 10` 设置的长。
以下三种情况认为主从复制超时：
* Replica 角度下，`repl-timeout` 时间内没有收到 Master 的 RDB 数据
* Replica 角度下，`repl-timeout` 时间内没有收到 Master 的数据或者 `PING`
* Master 角度下，`repl-timeout` 时间内没有收到 Replica 的 `REPLCONF ACK {offset}` 
# 6 `repl-disable-tcp-nodelay no`

是否配置开启 `TCP_NODELAY`：
* 如果开启，数据包立即发送，减少延迟
* 如果关闭，开启 Nagle 算法，会合并小数据包并等待确认（ACK）或数据积累，减少网络小包数量，但增加传输延迟
Redis 默认配置为开启 `TCP_NODELAY` ，减少了延迟，但增加了带宽，适用于实时应用。如果带宽成本高且需要远距离传输（比如跨地域主从）等，可容忍一定延迟，可设置为 `yes`。

# 7 `repl-backlog-size 1mb`

Replica 和 Master 断连后，Master 会把新的写入命令存到 `replication backlog buffer`，这个缓冲区的大小由这个配置决定，如果 Replica 重连后 `PSYNC` 发送的 `offset` 不在这个 buffer 中，则会进行全量同步。

计算公式： $（Master 写入命令速度 \times 命令大小 - 主从传输命令速度 \times 命令大小）\times 2$

# 8 `repl-backlog-ttl 3600`

Master 如果没有 Replica 连接了，`repl-backlog-ttl` 秒后将释放 `backlog`。

如果设置为 0 则表示永远不释放。
# 9 `replica-priority 100`

用于 `Sentinel` 选主，值越低则被选主的优先级越高，默认为 100，如果为 0 表示这个节点永远不会被选为 Master。

# 10 `propagation-error-behavior ignore`

配置处理命令和读取持久化的 AOF 文件时如果遇到错误的行为：
* ignore，表示忽略，继续处理后续。可能导致主从数据不一致，为了兼容旧版本的功能
* panic，立刻终止报错，人工介入修复
* panic-on-replicas，只有副本遇到错误时才会终止报错

# 11 `replica-ignore-disk-write-errors no`
如果从 Master 同步的命令无法持久化到副本磁盘，则直接崩溃，一般不建议修改。如果为了版本兼容可以修改为 `yes`。
# 12 .12 `replica-announced yes`
如果设置为 `no` 则该节点会忽略 `sentinel replicas <master>` 命令，且不会暴露给 Redis sentinel 的客户端。
哪怕设置为 `no`，该节点也是有可能被选为 Master，可设置 `replica-priority` 为 0 避免该节点被选为 Master。
# 13 `min-replicas-to-write 3` 和 `min-replicas-max-lag 10`
配置 Master 接受写命令的限制：
* 最小在线 Replica 数为 3
* Replica 最大延迟为 10 s
这两个配置可以显著降低主从数据不一致的问题。设置任何一个值为 0 可以关闭限制，默认为关闭。
# 14 `replica-announce-ip` 和 `replica-announce-port`

用于 NAT 网络中配置 Replica 的真正 IP 和端口，防止 Master 无法访问 Replica。

# 15 Reference
* [redis.conf](https://raw.githubusercontent.com/redis/redis/7.0/redis.conf)