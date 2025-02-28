---
title: "Nginx极简实战—Nginx服务器高性能优化配置，轻松实现10万并发访问量-阿里云开发者社区"
tags:
  - "clippings literature-note"
date: 2025-02-27
time: 2025-02-27T16:09:32+08:00
source: "https://developer.aliyun.com/article/791260"
---
版权声明：

本文内容由阿里云实名注册用户自发贡献，版权归原作者所有，阿里云开发者社区不拥有其著作权，亦不承担相应法律责任。具体规则请查看《 [阿里云开发者社区用户服务协议](https://developer.aliyun.com/article/768092)》和 《[阿里云开发者社区知识产权保护指引](https://developer.aliyun.com/article/768093)》。如果您发现本社区中有涉嫌抄袭的内容，填写 [侵权投诉表单](https://yida.alibaba-inc.com/o/right)进行举报，一经查实，本社区将立刻删除涉嫌侵权内容。

**简介：** 如何使Nginx轻松实现10万并发访问量。通常来说，一个正常的 Nginx Linux 服务器可以达到 500,000 – 600,000 次/秒 的请求处理性能，如果Nginx服务器经过优化的话，则可以稳定地达到 904,000 次/秒 的处理性能，大大提高Nginx的并发访问量。

前面讲了如何配置Nginx虚拟主机，如何配置服务日志等很多基础的内容。

今天要说的是Nginx服务器高性能优化的配置，如何使Nginx轻松实现10万并发访问量。

通常来说，一个正常的 Nginx Linux 服务器可以达到 500,000 – 600,000 次/秒 的请求处理性能，如果Nginx服务器经过优化的话，则可以稳定地达到 904,000 次/秒 的处理性能，大大提高Nginx的并发访问量。

这里需要特别说明的是：

1、本文中所有列出来的配置都是在我的测试环境验证的，你需要根据你服务器的情况进行配置。

2、Nginx的优化需要进行进行压力测试，这里压力测试用的是Apache ab测试工具，不熟悉的可以看看我之前的文章：《[如何使用apache ab性能测试工具进行压力测试](https://www.cnblogs.com/zhangweizhong/p/12349972.html)》

## 一、优化思路

我们知道，Nginx的压力主要来自于tcp连接数和文件打开数两个方面。

问题分析：nginx要成功响应请求，会有如下两个限制：

1、nginx接受的tcp连接多，能否建立起来?

2、nginx响应过程,要打开许多文件，能否打开?

所以，只要我们针对上面两个限制进行优化，就能大幅提升Nginx的效率。具体如下图所示：

![image.png](https://ucc.alicdn.com/pic/developer-ecology/ce0a5e6ada354860abeedbb24505634c.png?x-oss-process=image%2Fresize%2Cw_1400%2Fformat%2Cwebp "image.png")

## 二、优化方案

Nginx的工作模式如下图所示。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/8447d083faa84f42a38c2b688f35d152.png?x-oss-process=image%2Fresize%2Cw_1400%2Fformat%2Cwebp "image.png")

**一、优化步骤：**

1\. 找到Nginx服务器瓶颈。

2\. 优化配置。

3\. 重新压力测试

注意：在配置修改之后务必要进行压力测试，这样可以观测到具体是哪个配置修订的优化效果最明显。通过这种有效测试方法可以为你节省大量时间。

**二、找出Nginx的瓶颈**

1\. 打开Apache ab压力测试工具，输入如下命令：ab -n 200000 -c 5000 http://localhost:8080/index.html。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/1f5eb45f51b54d76ae15ba0ae20141b4.png?x-oss-process=image%2Fresize%2Cw_1400%2Fformat%2Cwebp "image.png")

2\. 查看Nginx 状态信息

在浏览器中输入nginx的地址：[http://127.0.0.1/status](http://127.0.0.1/status)，查看nginx的状态信息。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/7c20ec3e887842a3b147a72cb502066e.png?x-oss-process=image%2Fresize%2Cw_1400%2Fformat%2Cwebp "image.png")

注意查看connections，waiting等参数信息。从而确定如何优化相关参数。

Nginx 状态信息打开的方法，这里就不细说了，不清楚的可以看我之前的文章，《[Nginx总结（八）启用Nginx Status及状态参数详解](https://www.cnblogs.com/zhangweizhong/p/12347345.html)》

## 三、配置优化

根据上面的方法总结起来，一般来说nginx 配置文件中对优化比较有作用的为以下几项：

**Nginx优化配置项：**

1）优化 workprocess，cpu

```
worker_processes 8;      // 根据CPU核数配置
worker_cpu_affinity 00000001 00000010 00000100 00001000 00010000  00100000 01000000 10000000;
```

2）事件处理模型优化

nginx的连接处理机制在于不同的操作系统会采用不同的I/O模型，Linux下，nginx使用epoll的I/O多路复用模型，在freebsd使用kqueue的IO多路复用模型，在solaris使用/dev/pool方式的IO多路复用模型，在windows使用的icop等等。

```
events {
    worker_connections  10240;    // 
    use epoll;
}
```

要根据系统类型不同选择不同的事务处理模型，我们使用的是Centos，因此将nginx的事件处理模型调整为epoll模型。

说明：在不指定事件处理模型时，nginx默认会自动的选择最佳的事件处理模型服务。

3）设置work\_connections 连接数

```
worker_connections  10240;
```

4）每个进程的最大文件打开数

```
worker_rlimit_nofile 65535;  # 一般等于ulimit -n系统值
```

5）keepalive timeout会话保持时间

6）GZIP压缩性能优化

```
gzip on;       #表示开启压缩功能
gzip_min_length  1k; #表示允许压缩的页面最小字节数，页面字节数从header头的Content-Length中获取。默认值是0，表示不管页面多大都进行压缩，建议设置成大于1K。如果小于1K可能会越压越大
gzip_buffers     4 32k; #压缩缓存区大小
gzip_http_version 1.1; #压缩版本
gzip_comp_level 6; #压缩比率， 一般选择4-6，为了性能
gzip_types text/css text/xml application/javascript;  #指定压缩的类型 gzip_vary on;　#vary header支持
```

7）proxy超时设置

```
proxy_connect_timeout 90;
proxy_send_timeout  90;
proxy_read_timeout  4k;
proxy_buffers 4 32k;
proxy_busy_buffers_size 64k
```

8）高效传输模式

```
sendfile on; # 开启高效文件传输模式。
tcp_nopush on; #需要在sendfile开启模式才有效，防止网路阻塞，积极的减少网络报文段的数量。将响应头和正文的开始部分一起发送，而不一个接一个的发送。
```

## 四、系统优化

Nginx要达到最好的性能，出了要优化Nginx服务本身之外，还需要在nginx的服务器上的内核参数。Linux系统内核层面进行相关优化。例如：

1）调节系统同时发起的tcp连接数

`net.core.somaxconn = 262144`

2）允许等待中的监听

`net.core.somaxconn = 4096`

3） tcp连接快速回收

4） tcp连接重用  


```
net.ipv4.tcp\_tw\_recycle = 1

net.ipv4.tcp\_tw\_reuse = 1  
```


5）不抵御洪水攻击


```
net.ipv4.tcp\_syncookies = 0  

net.ipv4.tcp\_max\_orphans = 262144  #该参数用于设定系统中最多允许存在多少TCP套接字不被关联到任何一个用户文件句柄上，主要目的为防止Ddos攻击
```


6）最大文件打开数

`ulimit -n 30000`

需要注意的是：这些参数追加到/etc/sysctl.conf,然后执行sysctl -p 生效。

## 最后

以上，就把Nginx服务器高性能优化的配置介绍完了，大家可以根据我提供的方法，每个参数挨个设置一遍，看看相关的效果。这些都是一点点试出来的，这样才能更好的理解各个参数的意义。