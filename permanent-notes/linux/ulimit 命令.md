---
title: ulimit 命令
tags:
  - permanent-note
  - linux/command
date: 2025-03-04
time: 10:53
aliases:
---
`ulimit` 是一个在 Unix 和类 Unix 操作系统（如 Linux）中使用的命令，它为用户限制程序可以使用的资源量。这些限制对于防止程序错误地消耗过多系统资源非常重要，有助于维护系统的稳定性和安全性。

### 使用方法

通过 `ulimit` 命令，你可以查看或设置当前 shell 会话的资源限制。以下是该命令的一些基本用法：

- 查看所有限制：`ulimit -a`
- 设置文件大小限制（单位：KB）：`ulimit -f [size]`
- 设置最大进程数：`ulimit -u [number]`
- 设置最大打开文件描述符数量：`ulimit -n [number]`

### 软限制与硬限制

`ulimit` 分为软限制和硬限制：

- **软限制**：这是实际的限制值，任何进程都可以将其降低至任意非负值，但不能增加。
- **硬限制**：这是上限值，代表软限制的最大值。只有特权用户（通常是 root 用户）才能修改硬限制。

要分别查看或设置软限制和硬限制，可以使用 `-S` 和 `-H` 选项：

- 查看软限制：`ulimit -S`
- 查看硬限制：`ulimit -H`
- 设置软限制：`ulimit -S [option] [limit]`
- 设置硬限制：`ulimit -H [option] [limit]`

例如，查看当前用户的最大文件大小软限制：`ulimit -S -f`；设置最大文件大小硬限制为 1000 KB：`ulimit -H -f 1000`。

请注意，`ulimit` 的具体行为可能会根据不同的操作系统发行版和版本有所不同。如果你需要对系统资源进行严格的控制或者调整，请查阅相关文档或手册页 (`man ulimit`) 获取更详细的信息。

### 永久修改

```shell
[root@VM-4-7-centos ~]# cat /etc/security/limits.conf
# /etc/security/limits.conf
#
#This file sets the resource limits for the users logged in via PAM.
#It does not affect resource limits of the system services.
#
#Also note that configuration files in /etc/security/limits.d directory,
#which are read in alphabetical order, override the settings in this
#file in case the domain is the same or more specific.
#That means for example that setting a limit for wildcard domain here
#can be overriden with a wildcard setting in a config file in the
#subdirectory, but a user specific setting here can be overriden only
#with a user specific setting in the subdirectory.
#
#Each line describes a limit for a user in the form:
#
#<domain>        <type>  <item>  <value>
#
#Where:
#<domain> can be:
#        - a user name
#        - a group name, with @group syntax
#        - the wildcard *, for default entry
#        - the wildcard %, can be also used with %group syntax,
#                 for maxlogin limit
#
#<type> can have the two values:
#        - "soft" for enforcing the soft limits
#        - "hard" for enforcing hard limits
#
#<item> can be one of the following:
#        - core - limits the core file size (KB)
#        - data - max data size (KB)
#        - fsize - maximum filesize (KB)
#        - memlock - max locked-in-memory address space (KB)
#        - nofile - max number of open file descriptors
#        - rss - max resident set size (KB)
#        - stack - max stack size (KB)
#        - cpu - max CPU time (MIN)
#        - nproc - max number of processes
#        - as - address space limit (KB)
#        - maxlogins - max number of logins for this user
#        - maxsyslogins - max number of logins on the system
#        - priority - the priority to run user process with
#        - locks - max number of file locks the user can hold
#        - sigpending - max number of pending signals
#        - msgqueue - max memory used by POSIX message queues (bytes)
#        - nice - max nice priority allowed to raise to values: [-20, 19]
#        - rtprio - max realtime priority
#
#<domain>      <type>  <item>         <value>
#

#*               soft    core            0
#*               hard    rss             10000
#@student        hard    nproc           20
#@faculty        soft    nproc           20
#@faculty        hard    nproc           50
#ftp             hard    nproc           0
#@student        -       maxlogins       4

# End of file
* soft nofile 100001
* hard nofile 100002
root soft nofile 100001
root hard nofile 100002
* soft nofile 100001
* hard nofile 100002
root soft nofile 100001
root hard nofile 100002
* soft memlock unlimited
* hard memlock unlimited
```



需要修改的是 `nofile` 这两个配置，`hard` 表示当前可以设置的最大值，`soft` 表示当前的值，根据实际情况设置。

> 注意，nofile的最大值不能超过`/proc/sys/fs/nr_open`这个值，默认是(1048576)