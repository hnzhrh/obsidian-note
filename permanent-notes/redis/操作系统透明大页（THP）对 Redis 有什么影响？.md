---
title: 操作系统透明大页（THP）对 Redis 有什么影响？
tags:
  - permanent-note
  - middleware/redis/performance
date: 2025-03-06
time: 15:26
aliases:
---
# 1 THP 的机制
- **目标**：自动将多个 **4 KB 的小内存页** 合并为 **2 MB 的大页**，减少内存页表项数量，提升大内存应用的访问效率（如科学计算、大数据处理）。
- **行为**：
  - 操作系统 **动态合并** 小页为大页。
  - 当大页中的部分内存被修改时，触发 **拆分（Split）**，将大页退化为多个小页。

# 2 THP 如何加剧 Redis 内存碎片
## 2.1 与 Redis 内存分配模式的冲突

- Redis 使用 `jemalloc` 分配器，频繁申请/释放 **不同大小的内存块**（如几十字节的键、几 KB 的值）。
- **问题**：
  - THP 尝试将小页合并为大页，但 Redis 的 **细碎内存分配** 导致大页频繁 **拆分**。
  - 拆分后的内存区域变得 **不连续**，后续难以合并，形成碎片。

## 2.2 Copy-on-Write（写时复制）的副作用

- **场景**：Redis 执行 RDB 快照、AOF 重写或主从复制时，会创建子进程，父子进程通过 **共享内存** 节省资源。修改内存时触发 **写时复制**。
- **问题**：
  - 若父进程的内存被 THP 合并为 2 MB 大页，子进程修改 **任意 1 字节**，整个 2 MB 大页会被复制。
  - 导致 **额外内存占用** 和 **物理内存碎片**（复制后的新页可能分散在不同位置）。

## 2.3 延迟与不可预测性
- THP 的合并和拆分由操作系统 **后台异步完成**，难以预测。
- **后果**：
  - Redis 可能出现 **性能抖动**（如延迟上升）。
  - 内存碎片率（`mem_fragmentation_ratio`）波动增大。

# 3 禁用 THP 的优化效果

## 3.1 减少内存页拆分

- 禁用 THP 后，操作系统仅使用 **固定 4 KB 小页**，避免大页拆分导致的物理内存不连续。
- **结果**：内存分配更紧凑，外部碎片减少。

## 3.2 降低写时复制的开销

- 子进程修改数据时，仅复制 **实际修改的 4 KB 小页**，而非整个 2 MB 大页。
- **结果**：
  - 节省内存（减少复制量）。
  - 新复制的页更容易被紧凑利用，减少碎片。

## 3.3 提升内存分配器效率**

- `jemalloc` 等分配器针对小页内存管理优化，禁用 THP 后，内存分配策略与操作系统页管理 **一致**。
- **结果**：分配器更高效复用内存块，减少内部碎片。

# 4 如何禁用？

## 4.1 操作系统手动禁用（6.2 版本之前）

```bash
# 临时禁用（重启失效）
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 永久禁用（修改 GRUB 配置）
在 `/etc/default/grub` 中添加：
GRUB_CMDLINE_LINUX="transparent_hugepage=never"
update-grub  # 更新配置后重启

# 验证
cat /sys/kernel/mm/transparent_hugepage/enabled
# 输出应为 "always madvise [never]"
```

## 4.2 通过配置禁用

```config
#################### KERNEL transparent hugepage CONTROL ######################

# Usually the kernel Transparent Huge Pages control is set to "madvise" or
# or "never" by default (/sys/kernel/mm/transparent_hugepage/enabled), in which
# case this config has no effect. On systems in which it is set to "always",
# redis will attempt to disable it specifically for the redis process in order
# to avoid latency problems specifically with fork(2) and CoW.
# If for some reason you prefer to keep it enabled, you can set this config to
# "no" and the kernel global to "always".

disable-thp yes
```
# 5 Reference