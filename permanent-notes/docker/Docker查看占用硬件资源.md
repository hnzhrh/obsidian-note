---
title: Docker查看占用硬件资源
tags:
  - permanent-note
  - docker
date: 2025-01-15
time: 20:24
aliases:
---

要查看 Docker 占用的硬件资源，可以通过以下几种方法来获取详细信息。Docker 的资源使用通常包括 CPU、内存、磁盘空间等。以下是一些常用的命令和方法来查看这些资源占用情况：

### 1. 使用 `docker stats` 命令

`docker stats` 命令可以实时查看容器的资源使用情况，包括 CPU 使用率、内存使用、网络流量和磁盘 I/O 等。

#### 示例：

```bash
docker stats
```

此命令会列出所有正在运行的容器的资源使用情况，如下所示：

```
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O       BLOCK I/O     PIDS
123abc456def   mynginx   0.01%     50.2MiB / 1GiB      4.91%     1.5MB / 2.0MB  10MB / 5MB    5
```

每个列的解释：

- `CPU %`：容器使用的 CPU 百分比。
- `MEM USAGE / LIMIT`：容器使用的内存和容器的内存限制（如果设置了限制）。
- `MEM %`：内存使用的百分比。
- `NET I/O`：容器的网络输入/输出。
- `BLOCK I/O`：容器的磁盘输入/输出。
- `PIDS`：容器内进程数。

### 2. 使用 `docker system df` 查看 Docker 占用的磁盘空间

`docker system df` 命令可以查看 Docker 容器、镜像、卷等占用的磁盘空间。

#### 示例：

```bash
docker system df
```

命令输出会显示容器、镜像、卷等占用的空间，如下所示：

```
TYPE            TOTAL     ACTIVE    SIZE    RECLAIMABLE
Images          3         2         1.25GB  500MB (40%)
Containers      5         3         350MB   100MB (30%)
Local Volumes   2         2         1.2GB   400MB (33%)
Build Cache     0         0         0B      0B
```

每个字段的解释：

- `TYPE`：资源类型（容器、镜像、卷、构建缓存等）。
- `TOTAL`：资源的总数量。
- `ACTIVE`：当前活跃的资源数量。
- `SIZE`：资源占用的磁盘空间。
- `RECLAIMABLE`：可以回收的空间，表示可以删除的未使用的资源。

### 3. 使用 `docker info` 查看 Docker 运行时的系统资源

`docker info` 命令可以提供 Docker 守护进程的详细信息，包括主机的资源使用情况。

#### 示例：

```bash
docker info
```

输出中会包含主机资源的详细信息，例如：

```
Containers: 12
 Running: 5
 Paused: 0
 Stopped: 7
Images: 20
Server Version: 20.10.7
Storage Driver: overlay2
Backing Filesystem: ext4
Cgroup Driver: cgroupfs
Cgroup Version: 1
CPU: 8
Total Memory: 16GiB
Name: my-docker-host
ID: QNTA:HF56:JK79:...
Runtimes: runc
Default Runtime: runc
```

关键字段：

- `Total Memory`：主机的总内存。
- `CPU`：主机的 CPU 核心数。
- `Storage Driver`：使用的存储驱动（例如 `overlay2`）。
- `Backing Filesystem`：支持的文件系统（如 `ext4`）。
- `Cgroup Driver`：容器使用的 cgroup 驱动程序。

### 4. 使用系统命令查看 CPU 和内存使用情况

你还可以使用一些系统命令来查看主机上的硬件资源占用，特别是对于没有 `docker stats` 的场景（例如在没有 Docker 环境的情况下）：

#### 查看 CPU 使用情况：

```bash
top
```

或者使用 `htop`，它提供更友好的交互界面：

```bash
htop
```

#### 查看内存使用情况：

```bash
free -h
```

### 5. 查看 Docker 容器的资源分配

如果你想查看 Docker 容器具体的资源分配情况（如内存限制、CPU 限制等），可以使用 `docker inspect` 命令：

#### 示例：

```bash
docker inspect <container_id> | grep -i "Memory"
```

这将显示该容器的内存限制等资源配置。

### 6. 查看容器日志（如果有资源问题）

有时容器可能会因为资源超出限制而出现错误，可以查看容器的日志以帮助诊断问题。

#### 示例：

```bash
docker logs <container_id>
```

如果容器因资源限制被杀死或异常退出，日志中可能会有相关信息。

### 总结：

- 使用 `docker stats` 可以实时查看各个容器的资源使用情况。
- 使用 `docker system df` 查看 Docker 镜像、容器、卷占用的磁盘空间。
- 使用 `docker info` 查看 Docker 守护进程的硬件资源（如内存、CPU、磁盘）。
- 使用 `top` 或 `htop` 等系统工具查看主机资源使用情况。
- 使用 `docker inspect` 查看容器的资源分配。

这些方法可以帮助你全面了解 Docker 及其容器在主机上占用的硬件资源。

# Reference