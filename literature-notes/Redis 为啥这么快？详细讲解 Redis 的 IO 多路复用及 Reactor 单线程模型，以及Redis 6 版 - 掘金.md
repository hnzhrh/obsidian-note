---
title: "Redis 为啥这么快？详细讲解 Redis 的 I/O 多路复用及 Reactor 单线程模型，以及Redis 6 版 - 掘金"
tags:
  - "clippings literature-note"
date: 2025-04-23
time: 2025-04-23T07:46:00+08:00
source: "https://juejin.cn/post/7063104618007363614"
is_archie: false
---
![横幅](https://p9-piu.byteimg.com/tos-cn-i-8jisjyls3a/00f82abd1a4a48b7b01fcad2b753bcb1~tplv-8jisjyls3a-image.image) ![](https://p6-piu.byteimg.com/tos-cn-i-8jisjyls3a/f28400f482284bbebeec89ea9621b0b9~tplv-8jisjyls3a-image.image)

## Redis 版本迭代改进

**Redis 3.x单线程时代但性能依旧很快的主要原因**

1. 基于内存操作：所有数据都存于内存中，读写速度非常快，内存的响应时长约为100纳秒，运算都是内存级别的，因此性能比较高；
2. 数据结构简单：常用的数据结构中有些是专门设计的，如采用自己设计的简单动态字符串（Simple Dynamic String）作为字符串对象的底层数据结构，将获取字符串长度的时间复杂度提高到O(1)等特点；
3. I/O多路复用：使用 I/O多路复用功能来监听多个 socket 连接客户端，也就能够使用一个线程来处理多个请求，达到并发处理请求的效果；
4. 避免上下文切换：单线程模型，避免了多线程切换和多线程竞争带来的时间和性能上的消耗，并且避免了死锁问题的发生。

**Redis 4.0 之前一直采用单线程的主要原因有以下三个：**

1. 使用单线程模型，开发和维护简单；
2. 通过使用非阻塞的I/O多路复用模型，能够并发的处理多客户端的请求；
3. Redis 主要的性能瓶颈是 memory 或者 network bound 而并非 CPU。（最主要，也是决定性原因）

**Redis 4.0 针对删除操作引入多线程的原因：**

正常情况下，也就是针对小数据两量对象的删除是没有问题的，但是当删除的对象非常大，如几十兆的对象，短时间无法释放内存空间，此时，单线程删除就会阻塞到其它读写操作的进行。

因此，很自然的想法是不让主线程再处理删除操作了，开启一个异步线程去慢慢的删除对象就好了。Redis 4.0 引入了 `unlink key` 、 `flushall async` 、 `FLUSHDB ASYNC` 等命令， 对删除操作进行懒处理，主要用于 Redis 数据的异步删除。其处理读写请求的仍然只有一个线程，所以仍然称Redis为单线程的。（Redis之父一开始还是想用单线程去处优化，但是在系统繁忙时性能下降幅度较大，因此才使用了异步线程的方法）

**Redis 6.0 引入多线程提高网络I/O性能的原因：**

随着互联网业务系统的流量越来越大，Redis的网络I/O性能瓶颈越来越明显；单线程不能充分利用现代计算机多核CPU的优势，虽然官方建议多开几个实例就可以利用多核。

因此，在 Redis 6.0 中引入了 I/O 多线程的读写，主要思想是主线程不再进行 socket 的数据读写，而是开启一组独立的线程去监控处理 socket 的数据读写和请求解析等，达到并行化读写效果，这样就可以更加高效的并发的处理网络I/O。其中读写过程是多线程的，命令的执行实际上还是单线程处理。

在 **Redis6.0** 中，多线程机制默认是关闭的，如果需要使用多线程功能，需要在redis.conf中进行配置：1 `io-thread-do-reads yes` ，表示启动多线程。2 `io-thread 3` 设置线程个数。官方的建议是 4 核的 CPU建议线程数设置为 2 或 3，8 核 CPU 建议设置为 6，线程数一定要小于机器核数，线程数并不是越大越好。

## I/O 多路复用模型的发展

Unix网络编程中共包含五种IO模型，分别是Blocking IO（阻塞IO）、NoneBlocking IO（非阻塞IO）、IO multiplexing（IO多路复用）、signal driven IO（信号驱动IO）、asynchronous IO（异步IO）。Redis使用IO多路复用模型实现，本文重点讲解阻塞IO发展为非阻塞IO，继而发展出IO多路复用模型的过程，至于信号驱动IO和异步IO，本文不作介绍。

### 基本概念

#### 用户空间和内核空间

首先说一下CPU指令集的概念，CPU指令集是实现软件执行硬件的指令的集合。CPU指令操作的不规范，会对整个系统产生重大影响。因此CPU指令集必须设置权限，如Intel厂商将CPU指令集的权限由高到低分为ring 0 、ring 1、ring 2、ring 3分为。Linux系统中通常只使用ring 0 、ring 3两种权限：ring 0 权限最高，可以使用所有CPU指令；ring 3 权限最低，仅能使用常规CPU指令，不能直接访问硬件和所有内存。（下文介绍的用户态和内核态可以简单理解为分别对应了ring 3 、ring 0权限的指令集。）好了，这里为什么是不能访问所有内存呢？

在内存资源上，若任由应用程序随意访问内存空间的任意位置，很可能由于应用程序的运行或错误造成系统崩溃，如删除所有内存数据操作。故应用程序访问任意内存空间是很危险的。因此，操作系统将内存划分为操作系统内核使用的内存空间和应用程序访问的内存空间，比如说Linux系统中内存为4G，其中高地址的1G内存是专由操作系统内核使用的，该部分内存就称为内核空间；低地址的3G内存由应用程序使用，被称为用户空间。但并不是说系统内核不能使用低地址的3G内存空间，即内核空间只能由系统内核使用，用户空间能被系统内核和应用程序使用。

#### 用户态和内核态

顾名思义，运行在用户空间时的进程或线程处于用户态，运行在内核空间时的进程或线程处于内核态。 用户态切换到内核态包含3种方式： 系统调用、软中断（如异常）、硬中断（如I/O操作）。

![1644302850726.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3239240aeac484e9ffca37b56e555c7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 为什么说用户态和内核态的切换开销大？

例如，由用户态切换为内核态时，需要记录用户态的状态如寄存器等上下文、复制用户态参数到内核态、一致性和安全性检查等。如是再切换为用户态，则需要复制数据到用户态，并且回复用户态上下文。这个过程是比较耗时的，因此说两态之间的切换开销大。

#### 同步与异步

针对被调用者（服务提供者）来说，用于描述被调用者返回结果的通知方式。

若是被调用者执行完成后再返回结果，这是调用者是在一直等待返回结果的，此时称为同步过程；若是被调用者在被请求后立即返回一个值（有可能程序没有执行，该值并不一定是程序执行的结果，但是先返回一个值），被调用者在执行结束后主动通知（如 Callback）调用者结果，称为异步。

#### 阻塞与非阻塞

针对调用者来说，用于描述调用者在等待结果时的状态。

若是调用者在服务调用后一直等待服务返回结果，这个时候当前线程被挂起，称为阻塞；若是调用者立即返回，不等待返回结果，当前线程继续进行，称为非阻塞。

### 阻塞IO（BIO）

![1643465859589.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e1feb1704b7a4353868bed540dcadc82~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

recvfrom()函数是一个系统调用，用于从套接字上接收数据，并捕获数据源的发送地址。

当用户进程调用recvfrom()后，整个进程会被阻塞。当内核数据准备好了，然后将数据从内核缓冲区拷贝到用户空间（用户内存缓冲区），然后返回结果。用户进程才解除阻塞状态，继续运行。

为什么说内核需要准备数据呢？因为对于网络IO来说，很多时候数据没有到达，比如，等待着客户端那边输入数据或者还没有收到一个完整的UDP包）。因此，数据被拷贝到操作系统内核空间（内核缓冲区）中是需要一个准备过程的。

**缺点：不支持多socket连接** ，因为整个过程是阻塞的，只有在当前socket连接释放后，才能连接下一个socket客户端。

**解决办法** ：

1. **多线程？** 每来一个socket连接，就要开辟一个线程，如果来十万个，那就要开辟十万个线程。该方法显然不可行。
2. **使用线程池** ？socket客户端连接少的情况下可以使用，能在一定程度上缓解了资源占用，但是用户量大的情况下，线程池大小不好估计，必须根据响应规模调整池的大小。如果过小，对外界的响应并不比没有池的时候效果好；过大，浪费内存空间。
3. **非阻塞式IO** ：把socket设置为非阻塞的，数据没有准备好时直接返回一个错误信息，表示数据noready。这样就不用开辟多个线程处理多socket连接了，就可以采用轮询的方式遍历socket连接询问数据准备好了么。这就是非阻塞式IO。

### 非阻塞IO（NIO）

![1643466291402.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35e7c693907d4254880440d47bb89938~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

每当客户端与服务端进行连接时，就将这个socket放入到一个数组中，主进程就会轮询发起IO系统调用，知道有数据准备好。

**缺点：轮询问题** ，轮询将会不断地询问内核，这个过程涉及到用户态和内核态的切换，开销很大，这将占用大量的CPU时间，系统资源利用率较低；若有十万个socket连接，每次循环就要遍历十万个socket，若只有10个socket有数据，也会遍历十万个socket，导致资源浪费，效率降低。

**解决办法：将轮询过程放到内核态，** 既然两态之间的开销大，占用大量的CPU时间，那就避免两态切换，将轮询过程放到内核态中，不再两态转换而是直接从内核获得结果，这就是IO多路复用。（但是该办法之解决了轮询导致的两态切换问题，针对‘海量’socket连接时其效率降低和资源浪费的问题并没有解决，后续中介绍IO多路复用中select、poll、epoll三种实现方式中只有epoll解决了'海量'socket连接问题）

### I/O多路复用

![1643466492610.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc5220cd1fec4d4ebc95392d610b64b3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

通过一个进程监视多个文件描述符（为方便理解，可以简单理解为socket连接），具体来说就是利用 select、poll、epoll 同时监视多个文件描述符的I/O事件的发生，一旦某个或多个描述符就绪，就从阻塞态中唤醒，于是程序就会轮询一遍所有的文件描述符（epoll 只轮询那些真正发生了 I/O 事件的描述符），依次处理就绪的描述符。所以，I/O 多路复用的特点是通过一个进程同时等待多个文件描述符其中的任意一个或多个进入就绪状态，阻塞就可以解除，返回结果进行处理。

那么select、poll、epoll是如何同时监视多个文件描述符的呢？就是将NIO的轮询过程都放在了内核态进行，也即select、poll、epoll的实现就是NIO描述的轮询过程。下面介绍IO多路复用的三种实现方式：select、poll、epoll

`（文件描述符（File descriptor）是计算机科学中的一个术语，是一个用于表述指向文件的引用的抽象化概念。文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。但是文件描述符这一概念往往只适用于UNIX、Linux这样的操作系统。）`

#### select

将用户传入的文件描述符数组拷贝到内核空间，select 函数监视这些文件描述符，直到有描述符就绪（ 有数据可读、可写、或者有except ） 或者超时 ，函数返回。当select函数返回后，遍历文件描述符，来找到就绪的描述符。

```c
c 代码解读复制代码int select(
    int maxfdpl, //监控的文件描述符的最大数加1
    fd_set *readset, //监控的读事件的文件描述符
    fd_set *writeset, //监控的写事件的文件描述符
    fd_set *exceptset, //监控的异常事件的文件描述符
    struct timeval *timeout //等待所指定描述符的任何一个就绪可花多长时间：永远等、等待一段固定时间、不等
) 
//通过以下四个宏进行设置fd_set（文件描述符的集合），fd_set底层是由bitmap实现
FD_ZERO(int fd, fd_set* fds)   // 清空集合
FD_SET(int fd, fd_set* fds)    // 将描述符加入集合
FD_ISSET(int fd, fd_set* fds)  // 判断描述符是否在集合中 
FD_CLR(int fd, fd_set* fds)    // 将描述符删除
```
```c
c 代码解读复制代码//使用示例
int fds[SIZE];
int maxfd;
fd_set readfds;
struct timeval timeout;
fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd,(struct sockaddr*)&servaddr,sizeof(servaddr))
listen(fd,LISTENQ);

for (i = 0; i < SIZE; i++) {  //将新的连接描述符添加到数组中，模拟SIZE个客户端连接
    fds[i] = accept(fd,(struct sockaddr*)&clint,&addrlen);
    if (fd[i] > maxfd) {  //找到最大的文件描述符
        maxfd = fd[i]
    }
}

while (1) {  
  FD_ZERO(&readfds);  //初始化比特位
  for(i = 0; i < SIZE; i++){  //将文件描述符加入到集合中
      FD_SET(fds[i],&readfds);
  }
  nfds = select(maxfd + 1, &readfds, null, null, &timeout);  // 当没有数据时会一直阻塞在这一行，当socket数据准备完成时，将readfds相应的位置置位
  for (i = 0; i < SIZE; i++) {  // 遍历所有fd，判断哪位被置位了，有无读写事件发生
    if (FD_ISSET(fds[i], &readfds)) {
        read()//这里处理read事件
    }
  }
}
```

**优点** ：select函数将NIO的轮询过程放在了内核态中进行，让内核态来遍历，避免两态的频繁切换，节省开销。

**缺点** ：

1. fd\_set类型底层是由bitmap实现的，最大1024位，也就是说，一个进程最多只能处理1024个客户端；
2. 每次调用select函数时，都需要将文件描述符数组拷贝到内核态，高并发场景下这样的拷贝的开销是很大的。
3. select 返回可读文件描述符的个数，并没有通知用户态哪些socket有数据，仍然需要O(n)的遍历；
4. 针对“海量”的文件描述符，每次调用select都会循环扫描所有的文件描述符，效率低且浪费资源。

#### poll

poll()和select()的唯一的不同之处在于poll()使用自定义结构体 **pollfd** 替代了bitmap， **pollfd** 中封装了文件描述符，并通过 **pollfd** 的event属性注册可读、可写、出错事件（ POLLIN/POLLRDNORM 、 POLLOUT/POLLWRNORM、 POLLERR ），最后把 **pollfd** 交给内核，当有读写事件触发的时候，将 **pollfd** 的revents置位并返回。通过轮询所有的 **pollfd** ，根据revent字段确定该文件描述符是否发生了事件。

```c
c 代码解读复制代码int poll(
    struct pollfd *fdarry,  //指向pollfd结构体数组的第一个元素的指针
    nfds_t nfds,  //队列的长度
    int timeout  //超时时间
）
    
struct pollfd {
    int fd;                         // 需要监视的文件描述符
    short events;                   // 需要内核监视的事件
    short revents;                  // 实际发生的事件
};
```
```c
c 代码解读复制代码//使用示例
struct pollfd pollfds[SIZE];
nfds_t nfds;
int timeout;
fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd,(struct sockaddr*)&servaddr,sizeof(servaddr))
listen(fd,LISTENQ);

for (i = 0; i < SIZE; i++) {  //将新的连接描述符添加到数组中，模拟SIZE个客户端连接
    pollfds[i].fd = accept(fd,(struct sockaddr*)&clint,&addrlen);
    pollfds[i].events = POLLIN;
}

while (1) {  
  poll(pollfds, nfds, timeout);  //将pollfd数组传入内核中判断是否有事件发生，若发生，将对应revents置位
  for (i = 0; i < SIZE; i++) {  //遍历数组，判断哪个pollfd被置位了
    if (pollfds[i].revents & POLLIN)) {
        pollfds[i].revents = 0;  //将revents置0
          read()//这里处理read事件
    }
  }
}
```

**优点** ：使用结构体pollfd代替select中的bitmap，没有了1024的限制，可以一次处理更多的文件描述符。

**缺点** ：

1. pollfds数组拷贝到内核态，高并发场景下这样的拷贝的开销是很大的；
2. poll并没有通知用户态哪一个socket有数据，仍然需要O(n)的遍历；
3. 针对“海量”的文件描述符，每次调用poll都会循环扫描所有的文件描述符，效率低且浪费资源。

#### epoll

首先要调用 `epoll_create(int size)` 建立一个epoll对象（该对象存在于内核中，也就是说红黑树和双端链表都存在于内核空间中）参数size是内核保证能够正确处理的最大fd数，多于这个最大数时内核可不保证效果。 但是在 Linux 2.6.8 就弱化了该参数，只要不为0即可，实现了动态的增加最大fd数。

调用 `epoll_ctl()` ，能够把文件标识符增删改到内核红黑树中，红黑树中的fd就是目前需要监控的fd；其次，向内存中断处理程序绑定一个回调函数，哪个文件标识符的事件发生，就将该文件标识符放到双链表（就绪链表）中。

`epoll_wait()` 当在监控的fd中有事件发生时，即当一个socket的数据到了，将数据拷贝到内核缓冲区后发起一个中断，通过回调函数将文件标识符放到就绪链表中，然后返回用户态的进程。

```c
c 代码解读复制代码int epoll_create(int size);  //内核初始化一个eventpoll对象
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);   //负责把文件标识符增删改到内核红黑树，增(EPOLL_CTL_ADD),删(EPOLL_CTL_DEL),改(EPOLL_CTL_MOD)
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);   //等待事件

struct eventpoll {
    struct rb_root  rbr;  //红黑树的根节点，这颗树中存储着所有添加的需要监控的文件标识符
    struct list_head rdlist;  //双链表中存放着事件产生的文件标识符
};
```
```c
c 代码解读复制代码//使用示例
fd = socket(AF_INET, SOCK_STREAM, 0);
bind(fd,(struct sockaddr*)&servaddr,sizeof(servaddr))
listen(fd,LISTENQ);
struct epoll_event events[SIZE];  //epoll_event数据结构与poll中类似，除了没有revents属性
int epfd = epoll_create(SIZE);  //内核中创建eventpoll对象

for (i = 0; i < SIZE; i++) {  //将新的文件描述符添加到eventpoll对象
    struct epoll_event events;
    events[i].data.fd = accept(fd,(struct sockaddr*)&clint,&addrlen);
    events[i].events = EPOLLIN;
    epoll_ctl(epfd,EPOLL_CTL_ADD,events[i].data.fd,&events); 
}

while (1) {  
  nfds = epoll_wait(epfd,events,20,0);  //当文件描述符就绪时，将文件描述符放到就绪链表中，返回就绪fd的个数
  for (i = 0; i < nfds; i++) {  //遍历就绪链表
      read()//这里处理read事件
  }
}
```

**优点** ：

1. 只进行一次内核态的文件标识符拷贝，也就是说，使用 `epoll_ctl()` 将新的fd拷贝至内核态中，后续调用 `epoll_wait()` 时不再拷贝，节省开销；
2. 用户进程遍历就绪链表就可，不用遍历所有文件标识符，时间复杂度降为O(1)；
3. IO效率不随文件标识符数目增加而线性下降，通过中断+回调函数+双链表解决了大并发下的socket连接问题。

**缺点** ：只存在于Linux中。

#### 总结

|  | select | poll | epoll |
| --- | --- | --- | --- |
| 数据结构 | bitmap | 数组 | 红黑树 |
| 最大支持连接数 | 1024 | 无上限 | 无上限 |
| fd拷贝 | 调用select时需拷贝fd数组到内核态 | 调用poll时需拷贝fd数组到内核态 | 调用epoll\_ctl时拷贝fd进内核并保存，调用epoll\_wait不再拷贝 |
| 进程时间复杂度 | O(n) | O(n) | O(1) |
| 底层操作方式 | 轮询 | 轮询 | 回调 |

## Redis 线程模型

### 单线程消息处理模型

![1643465082307.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbae4e1ad60947e79a75871995e8c986~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) Redis 核心网络模型采用 Reactor 的方式来实现网络事件处理器，该处理器被称为文件事件处理器。文件事件处理器使用I/O多路复用来监控多个socket，并根据socket目前执行的任务来为socket关联不同的事件处理器。它的组成结构为4部分：多个套接字、I/O多路复用程序、文件事件分派器、事件处理器。因为文件事件分派器以单线程方式运行的，所以Redis才叫单线程模型。

`什么是文件事件嘞？文件事件是对socket操作的一个抽象，每种socket的操作都对应着一种文件事件，如socket的连接、读、写、关闭分别对应着不同的文件事件。`

文件事件处理器使用I/O多路复用程序来同时监听多个socket，当被监听的套接字准备好执行连接应答、读取、写入、关闭等操作时，与操作相对应的文件事件就会产生，将所有产生事件的套接字都放到一个队列里面，然后通过这个队列，以有序（sequentially）、同步（synchronously）、每次一个套接字的方式向文件事件分派器传送套接字。当上一个套接字产生的事件被处理完毕之后（该套接字为事件所关联的事件处理器执行完毕），文件事件分派器继续处理下一个套接字。

下图是以源码处理流程对Redis线程模型进行展示。 `说明：实线箭头表示同一层级函数顺序执行，虚线表示函数内部运行流程。`

![222.jpg](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e17d1c8df2164cd791bde30214b4d6e6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

#### 文件事件处理流程

- 调用aeMain()函数，当eventLoop->stop为0时，一直循环执行beforesleep()和aeProcessEvents()函数；
- beforesleep()阶段
	- 遍历队列 clients\_pending\_write回写数据；
	- 若writeToClient()没写完，注册写回调函数sendReplyToClient等待下次继续写；
- aeProcessEvents()函数内部有4个重要阶段，分别是aeApiPoll、就绪文件事件执行阶段、执行时间事件阶段，下面按照进程执行顺序介绍这3个阶段：
	- aeApiPoll阶段
		- 等待事件发生，阻塞返回；（其中epoll\_wait()的参数timeout取决于beforesleep前对时间时间的检测）
	- 就绪文件事件执行阶段
		- 当aeApiPoll执行完毕后存在就绪的文件事件，遍历就绪的文件时间进行处理；
		- 若是可读事件，执行回调函数readQueryFromClient()读取socket数据到输入缓冲区，执行命令，命令执行之后，若有响应数据需回写，将 client 加入到待写出队列 clients\_pending\_write；
		- 若是可写事件，执行回调函数，writeToClient()将缓冲区的数据回写到client，若 client 的写出缓冲区中有残留数据，为 client 注册一个命令回复器 sendReplyToClient，等待回写；
	- 执行时间事件阶段
		- 若存在时间事件，则执行。

### 多线程消息处理流程

与单线程消息处理流程唯一的不同之处是主线程不再进行 socket 的数据读写，而是开启一组独立的线程去监控处理 socket 的数据读写和请求解析等，命令执行还是由主线程完成。因此，虽然添加了多线程处理网络I/O，但是还是属于单线程的Reactor模型，与Reactor多线程模型的区别这里不在介绍。

下图中，可先简单的将InitServerLast处理部分看作多线程，主线程是在aeMain中顺序进行。

![111.jpg](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03c066121fd2402ca6fdd8c9e897220a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?) `说明：实线箭头表示同一层级函数顺序执行，虚线表示函数内部运行流程。`

#### IO线程的初始化及执行逻辑

- Redis启动时执行初始化函数 initThreadedIO()，该函数根据配置的线程个数，执行多少次\*IOThreadMain()函数，意思是为每个线程初始化执行一次该函数；
- \*IOThreadMain()函数内部包含一个死循环；
	- 死循环内部首先轮询100万次，判断任务计数器中的任务个数，若为0，则等待主线程分配 I/O 任务；
	- 轮询后若没有任务（首次启动时会给当前线程加锁进入休眠状态），等待主线程分配任务之后解锁；
	- 当主线程分配任务后，遍历任务队列io\_threads\_list执行，若当前操作是写操作，则把 client 的写出缓冲区中的数据回写到客户端；若是读取操作，则调用readQueryFromClient()读取socket客户端数据到输入缓冲区中，解析第一条命令，但是不会执行命令；
	- 完成io\_threads\_list中的所有任务后，将任务计数器io\_threads\_pending置0。
```c
c 代码解读复制代码void *IOThreadMain(void *myid) {
    /* The ID is the thread number (from 0 to server.iothreads_num-1), and is
     * used by the thread to just manipulate a single sub-array of clients. */
    long id = (unsigned long)myid;
    char thdname[16];   

    snprintf(thdname, sizeof(thdname), "io_thd_%ld", id);
    redis_set_thread_title(thdname);
    redisSetCpuAffinity(server.server_cpulist);
    makeThreadKillable();

    while(1) {
        /* Wait for start */
        for (int j = 0; j < 1000000; j++) {
            if (getIOPendingCount(id) != 0) break;
        }

        /* Give the main thread a chance to stop this thread. */
        if (getIOPendingCount(id) == 0) {
            pthread_mutex_lock(&io_threads_mutex[id]);
            pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }

        serverAssert(getIOPendingCount(id) != 0);

        /* Process: note that the main thread will never touch our list
         * before we drop the pending count to 0. */
        listIter li;
        listNode *ln;
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        listEmpty(io_threads_list[id]);
        setIOPendingCount(id, 0);
    }
}
```

#### 文件事件处理流程

- 调用aeMain()函数，当eventLoop->stop为0时，一直循环执行aeProcessEvents()函数；
- aeProcessEvents()函数内部有4个重要阶段，分别是beforesleep、aeApiPoll、就绪文件事件执行阶段、执行时间事件阶段，下面按照进程执行顺序介绍这4个阶段：
	- beforesleep阶段
		- 遍历等待读取的client队列 clients\_pending\_read，将client轮询均匀地分配给 I/O 线程和主线程；
		- 根据分配的client个数设置每个线程的任务计数器io\_threads\_pending；
		- 轮询，直到所有线程的任务计数器都为0，表示所有线程任务结束；（此时开始等待上边的多线程执行的结果）
		- 遍历队列clients\_pending\_read，消除CLIENT\_PENDING\_READ标记，执行命令；
		- 命令执行之后，若有响应数据需回写，将 client 加入到待写出队列 clients\_pending\_write；
		- 遍历队列 clients\_pending\_write，轮询均匀地分配给 I/O 线程和主线程；
		- 轮询，直到所有线程的任务计数器都为0，表示所有线程任务结束；
		- 再遍历一次 clients\_pending\_write 队列，若 client 的写出缓冲区中有残留数据，为 client 注册一个命令回复器 sendReplyToClient，等待回写；
	- aeApiPoll阶段
		- 等待事件发生，阻塞返回；（其中epoll\_wait()的参数timeout取决于beforesleep前对时间时间的检测）
	- 就绪文件事件执行阶段
		- 当aeApiPoll执行完毕后存在就绪的文件事件，遍历就绪的文件时间进行处理；
		- 若是可读事件，执行回调函数readQueryFromClient()把 client 加入到 clients\_pending\_read 任务队列，交给 后续IO线程去处理；
		- 若是可写事件，执行回调函数writeToClient()将缓冲区的数据回写到client；
	- 执行时间事件阶段
		- 若存在时间事件，则执行。
```c
c 代码解读复制代码void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|
                                   AE_CALL_BEFORE_SLEEP|
                                   AE_CALL_AFTER_SLEEP);
    }
}
```
```c
c 代码解读复制代码/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * If flags is 0, the function does nothing and returns.
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * if flags has AE_DONT_WAIT set the function returns ASAP until all
 * the events that's possible to process without to wait are processed.
 * if flags has AE_CALL_AFTER_SLEEP set, the aftersleep callback is called.
 * if flags has AE_CALL_BEFORE_SLEEP set, the beforesleep callback is called.
 *
 * The function returns the number of events processed. */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want to call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        struct timeval tv, *tvp;
        int64_t usUntilTimer = -1;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            usUntilTimer = usUntilEarliestTimer(eventLoop);

        if (usUntilTimer >= 0) {
            tv.tv_sec = usUntilTimer / 1000000;
            tv.tv_usec = usUntilTimer % 1000000;
            tvp = &tv;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        if (eventLoop->flags & AE_DONT_WAIT) {
            tv.tv_sec = tv.tv_usec = 0;
            tvp = &tv;
        }

        if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
            eventLoop->beforesleep(eventLoop);

        /* Call the multiplexing API, will return only on timeout or when
         * some event fires. */
        numevents = aeApiPoll(eventLoop, tvp);

        /* After sleep callback. */
        if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
            eventLoop->aftersleep(eventLoop);

        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */

            /* Normally we execute the readable event first, and the writable
             * event later. This is useful as sometimes we may be able
             * to serve the reply of a query immediately after processing the
             * query.
             *
             * However if AE_BARRIER is set in the mask, our application is
             * asking us to do the reverse: never fire the writable event
             * after the readable. In such a case, we invert the calls.
             * This is useful when, for instance, we want to do things
             * in the beforeSleep() hook, like fsyncing a file to disk,
             * before replying to a client. */
            int invert = fe->mask & AE_BARRIER;

            /* Note the "fe->mask & mask & ..." code: maybe an already
             * processed event removed an element that fired and we still
             * didn't processed, so we check if the event is still valid.
             *
             * Fire the readable event if the call sequence is not
             * inverted. */
            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
            }

            /* Fire the writable event. */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert) {
                fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                if ((fe->mask & mask & AE_READABLE) &&
                    (!fired || fe->wfileProc != fe->rfileProc))
                {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```
```c
c 代码解读复制代码void beforeSleep(struct aeEventLoop *eventLoop) {
    /* We should handle pending reads clients ASAP after event loop. */
    handleClientsWithPendingReadsUsingThreads();
    ...
    /* Handle writes with pending output buffers. */
    handleClientsWithPendingWritesUsingThreads();
    ...
}

int handleClientsWithPendingReadsUsingThreads(void) {
    if (!server.io_threads_active || !server.io_threads_do_reads) return 0;
    int processed = listLength(server.clients_pending_read);
    if (processed == 0) return 0;

    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_READ;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    /* Also use the main thread to process a slice of clients. */
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;    
    }

    /* Run the list of clients again to process the new buffers. */
    while(listLength(server.clients_pending_read)) {
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read,ln);

        serverAssert(!(c->flags & CLIENT_BLOCKED));
        if (processPendingCommandsAndResetClient(c) == C_ERR) {
            /* If the client is no longer valid, we avoid
             * processing the client later. So we just go
             * to the next. */
            continue;
        }

        processInputBuffer(c);

        /* We may have pending replies if a thread readQueryFromClient() produced
         * replies and did not install a write handler (it can't).
         */
        if (!(c->flags & CLIENT_PENDING_WRITE) && clientHasPendingReplies(c))
            clientInstallWriteHandler(c);
    }

    /* Update processed count on server */
    server.stat_io_reads_processed += processed;

    return processed;
}

int handleClientsWithPendingWritesUsingThreads(void) {
    int processed = listLength(server.clients_pending_write);
    if (processed == 0) return 0; /* Return ASAP if there are no clients. */

    /* If I/O threads are disabled or we have few clients to serve, don't
     * use I/O threads, but the boring synchronous code. */
    if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
        return handleClientsWithPendingWrites();
    }

    /* Start threads if needed. */
    if (!server.io_threads_active) startThreadedIO();

    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;

        /* Remove clients from the list of pending writes since
         * they are going to be closed ASAP. */
        if (c->flags & CLIENT_CLOSE_ASAP) {
            listDelNode(server.clients_pending_write, ln);
            continue;
        }

        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_WRITE;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        setIOPendingCount(j, count);
    }

    /* Also use the main thread to process a slice of clients. */
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);
    }
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += getIOPendingCount(j);
        if (pending == 0) break;
    }

    /* Run the list of clients again to install the write handler where
     * needed. */
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        /* Install the write handler if there are pending writes in some
         * of the clients. */
        if (clientHasPendingReplies(c) &&
                connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    listEmpty(server.clients_pending_write);

    /* Update processed count on server */
    server.stat_io_writes_processed += processed;

    return processed;
}
```
```c
c 代码解读复制代码static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, j, numevents = 0;

    memcpy(&state->_rfds,&state->rfds,sizeof(fd_set));
    memcpy(&state->_wfds,&state->wfds,sizeof(fd_set));

    retval = select(eventLoop->maxfd+1,
                &state->_rfds,&state->_wfds,NULL,tvp);
    if (retval > 0) {
        for (j = 0; j <= eventLoop->maxfd; j++) {
            int mask = 0;
            aeFileEvent *fe = &eventLoop->events[j];

            if (fe->mask == AE_NONE) continue;
            if (fe->mask & AE_READABLE && FD_ISSET(j,&state->_rfds))
                mask |= AE_READABLE;
            if (fe->mask & AE_WRITABLE && FD_ISSET(j,&state->_wfds))
                mask |= AE_WRITABLE;
            eventLoop->fired[numevents].fd = j;
            eventLoop->fired[numevents].mask = mask;
            numevents++;
        }
    }
    return numevents;
}
```

## 参考

《Redis5设计与源码分析》

《Redis设计与实现》

《UNIX网络编程卷1：套接字联网API（第3版）》

[Redis 多线程网络模型全面揭秘](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F356059845 "https://zhuanlan.zhihu.com/p/356059845")

[redis 源码剖析和注释技术博客专栏](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fmenwenjun%2Fredis_source_annotation "https://github.com/menwenjun/redis_source_annotation")

[彻底搞懂Reactor模型和Proactor模型](https://link.juejin.cn/?target=https%3A%2F%2Fcloud.tencent.com%2Fdeveloper%2Farticle%2F1488120 "https://cloud.tencent.com/developer/article/1488120")

[Redis 6.0 多线程IO处理过程详解](https://link.juejin.cn/?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F144805500%3Futm_source%3Dwechat_session%26utm_medium%3Dsocial%26utm_oi%3D1304940325903556608 "https://zhuanlan.zhihu.com/p/144805500?utm_source=wechat_session&utm_medium=social&utm_oi=1304940325903556608")

`如有侵权，请联系删除；如有错误，请批评指正。`

评论 0

暂无评论数据

![](https://lf-web-assets.juejin.cn/obj/juejin-web/xitu_juejin_web/c12d6646efb2245fa4e88f0e1a9565b7.svg) 7

![](https://lf-web-assets.juejin.cn/obj/juejin-web/xitu_juejin_web/336af4d1fafabcca3b770c8ad7a50781.svg) 评论

![](https://lf-web-assets.juejin.cn/obj/juejin-web/xitu_juejin_web/3d482c7a948bac826e155953b2a28a9e.svg) 收藏

APP内打开