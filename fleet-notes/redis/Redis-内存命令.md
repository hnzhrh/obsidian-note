---
title: Redis-内存命令
tags:
  - fleet-note
  - middleware/redis/ops
date: 2024-11-30
time: 11:26
aliases:
---
# 查看指定 Key 的占用内存大小
在 Redis 中，你可以使用 `MEMORY USAGE` 命令来查看一个特定 key 的内存占用情况。这个命令会返回给定 key 所消耗的字节数。

使用方法如下：

```redis
MEMORY USAGE <key>
```

例如，如果你想要查看名为 `mykey` 的键占用了多少内存，可以执行以下命令：

```redis
MEMORY USAGE mykey
```

这条命令将返回 `mykey` 键所占用的内存大小（以字节为单位）。需要注意的是，`MEMORY USAGE` 返回的值包括了键本身以及其关联值的内存开销，还包括一些额外的元数据开销。

此外，Redis 还提供了 `MEMORY STATS` 和 `MEMORY PURGE` 等其他与内存管理相关的命令，但这些命令通常用于更高级的内存分析和优化工作。对于仅仅想了解某个键占用内存大小的情况来说，`MEMORY USAGE` 是最直接的选择。

如果你正在使用 Redis 的版本较旧，并且不支持 `MEMORY USAGE` 命令，你可能需要考虑升级到更新版本的 Redis，或者使用其他间接的方法来估计内存使用情况，比如通过 `INFO` 命令获取整体内存使用情况然后进行估算。不过，从 Redis 4.0 开始，`MEMORY USAGE` 命令就已经被引入并广泛使用了。

# References