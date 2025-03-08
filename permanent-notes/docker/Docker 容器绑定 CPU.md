---
title: Docker 容器绑定 CPU
tags:
  - permanent-note
  - docker
date: 2025-03-06
time: 16:46
aliases:
---
在 Docker 中，你可以通过使用 `--cpuset-cpus` 参数来绑定容器到特定的 CPU 核。这允许你指定容器运行时可以使用的 CPU 核心。这对于确保容器只使用特定的 CPU 资源特别有用，尤其是在多核系统上。

以下是一些具体的用法示例：

1. **绑定单个 CPU 核心**：如果你想要将一个容器绑定到一个特定的核心，比如第 1 核（注意，在大多数系统中，CPU 编号是从 0 开始的），你可以这样启动你的容器：
   ```shell
   docker run -it --cpuset-cpus="0" ubuntu
   ```
   这条命令会启动一个新的 Ubuntu 容器，并将其限制在第一个 CPU 核心上运行。

2. **绑定多个 CPU 核心**：如果你想让容器能够使用多个 CPU 核心，比如说第 1 和第 2 核心，你可以这样做：
   ```shell
   docker run -it --cpuset-cpus="0,1" ubuntu
   ```
   或者，如果你想指定一个范围，例如从第 0 到第 2 核心，可以使用短划线 (-) 表示范围：
   ```shell
   docker run -it --cpuset-cpus="0-2" ubuntu
   ```

3. **检查容器的 CPU 亲和性设置**：你可以使用 `docker inspect` 命令查看已运行容器的详细信息，包括其 CPU 亲和性设置。例如：
   ```shell
   docker inspect <container_id> | grep CpusetCpus
   ```
   将 `<container_id>` 替换为你的容器的实际 ID 或名称。

请注意，尽管指定了 CPU 核心，Docker 容器内的进程仍然需要支持并正确配置以利用这些设置。此外，操作系统层面的调度也会影响最终性能和资源分配效果。因此，在生产环境中进行此类操作之前，请确保充分测试并了解潜在的影响。

# Reference
* [Docker容器CPU资源管理：多核CPU内核绑定与份额调整-CSDN博客](https://blog.csdn.net/qq_39517116/article/details/134007507)
* [笔记：Docker容器的资源控制之CPU](https://zhuanlan.zhihu.com/p/668639799)
* [tasksetCPU亲和力&docke容器资源限制 - Mr&Yu - 博客园](https://www.cnblogs.com/bixiaoyu/p/9339151.html)
