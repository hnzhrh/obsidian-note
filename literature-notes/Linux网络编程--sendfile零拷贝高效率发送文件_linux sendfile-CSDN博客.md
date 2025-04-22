---
title: "Linux网络编程--sendfile零拷贝高效率发送文件_linux sendfile-CSDN博客"
tags:
  - "clippings literature-note"
date: 2025-04-11
time: 2025-04-11T13:57:42+08:00
source: "https://blog.csdn.net/hnlyyk/article/details/50856268"
is_archie: false
---
本博文介绍使用sendfile函数进行零拷贝发送文件，实现高效数据传输，并对其进行验证。

那么什么是sendfile呢？

linux系统使用man sendfile,查看sendfile原型如下：

#include <sys/sendfile.h>

ssize\_t sendfile(int out\_fd, int in\_fd, off\_t \*offset, size\_t count);

**参数特别注意的是：in\_fd必须是一个支持mmap函数的文件描述符，也就是说必须指向真实文件，不能使socket描述符和管道。**

**out\_fd必须是一个socket描述符。**

**由此可见sendfile几乎是专门为在网络上传输文件而设计的。**

Sendfile 函数在两个文件描述符之间直接传递数据(完全在内核中操作，传送)，从而避免了内核缓冲区数据和用户缓冲区数据之间的拷贝，操作效率很高，被称之为零拷贝。  

传统方式read/write send/recv  
在传统的文件传输里面（read/write方式），在实现上其实是比较复杂的，需要经过多次上下文的切换，我们看一下如下两行代码：  
1\. read(file, tmp\_buf, len);

2\. write(socket, tmp\_buf, len);

以上两行代码是传统的read/write方式进行文件到socket的传输。

当需要对一个文件进行传输的时候，其具体流程细节如下：

1、调用read函数，文件数据被copy到内核缓冲区

2、read函数返回，文件数据从内核缓冲区copy到用户缓冲区

3、write函数调用，将文件数据从用户缓冲区copy到内核与socket相关的缓冲区。

4、数据从socket缓冲区copy到相关协议引擎。

以上细节是传统read/write方式进行网络文件传输的方式，我们可以看到，

在这个过程当中，文件数据实际上是经过了四次copy操作： 硬盘—>内核buf—>用户buf—>socket相关缓冲区(内核)—>协议引擎  

  

新方式sendfile

sendfile系统调用则提供了一种减少以上多次copy，提升文件传输性能的方法。

1、系统调用 sendfile() 通过 DMA 把硬盘数据拷贝到 kernel buffer，然后数据被 kernel 直接拷贝到另外一个与 socket 相关的 kernel buffer。这里没有 user mode 和 kernel mode 之间的切换，在 kernel 中直接完成了从一个 buffer 到另一个 buffer 的拷贝。  
2、DMA 把数据从 kernel buffer 直接拷贝给协议栈，没有切换，也不需要数据从 user mode 拷贝到 kernel mode，因为数据就在 kernel 里。

  

服务端：

```cpp
#include <sys/socket.h>

#include <netinet/in.h>

#include <arpa/inet.h>

#include <assert.h>

#include <stdio.h>

#include <unistd.h>

#include <stdlib.h>

#include <errno.h>

#include <string.h>

#include <sys/types.h>

#include <sys/stat.h>

#include <fcntl.h>

#include <sys/sendfile.h>

 

int main( int argc, char* argv[] )

{

    if( argc <= 3 )

    {

        printf( "usage: %s ip_address port_number filename\n", basename( argv[0] ) );

        return 1;

    }

    longnum=0,sum=0;

    static char buf[1024];

    memset(buf,'\0',sizeof(buf));

    const char* ip = argv[1];

    int port = atoi( argv[2] );

    const char* file_name = argv[3];

 

    int filefd = open( file_name, O_RDONLY );

    assert( filefd > 0 );

    struct stat stat_buf;

    fstat( filefd, &stat_buf );

        

        FILE *fp=fdopen(filefd,"r");

        

    struct sockaddr_in address;

    bzero( &address, sizeof( address ) );

    address.sin_family = AF_INET;

    inet_pton( AF_INET, ip, &address.sin_addr );

    address.sin_port = htons( port );

 

    int sock = socket( PF_INET, SOCK_STREAM, 0 );

    assert( sock >= 0 );

 

    int ret = bind( sock, ( struct sockaddr* )&address, sizeof( address ) );

    assert( ret != -1 );

 

    ret = listen( sock, 5 );

    assert( ret != -1 );

 

    struct sockaddr_in client;

    socklen_t client_addrlength = sizeof( client );

    int connfd = accept( sock, ( struct sockaddr* )&client, &client_addrlength );

    if ( connfd < 0 )

    {

        printf( "errno is: %d\n", errno );

    }

    else

    {

        time_t begintime=time(NULL);

        

        while((fgets(buf,1024,fp))!=NULL){

            num=send(connfd,buf,sizeof(buf),0);

            sum+=num;

            memset(buf,'\0',sizeof(buf));

        }

        

//        sendfile( connfd, filefd, NULL, stat_buf.st_size );

        time_t endtime=time(NULL);

        printf("sum:%ld\n",sum);

        printf("need time:%d\n",endtime-begintime);

        close( connfd );

    }

 

    close( sock );

    return 0;

}
```
  
客户端：
```cpp
#include <sys/socket.h>

#include <netinet/in.h>

#include <arpa/inet.h>

#include <assert.h>

#include <stdio.h>

#include <unistd.h>

#include <stdlib.h>

#include <errno.h>

#include <string.h>

#include <sys/types.h>

#include <sys/stat.h>

#include <fcntl.h>

#include <sys/sendfile.h>

 

int main( int argc, char* argv[] )

{

    if( argc <= 3 )

    {

        printf( "usage: %s ip_address port_number filename\n", basename( argv[0] ) );

        return 1;

    }

    static char buf[1024];

    memset(buf,'\0',sizeof(buf));

    const char* ip = argv[1];

    int port = atoi( argv[2] );

    const char* file_name = argv[3];

 

    int filefd = open( file_name, O_WRONLY );

    if(filefd<=0)

        printf("open error:%s",strerror(errno));

    assert( filefd > 0 );

        

        FILE *fp=fdopen(filefd,"w");

        

    struct sockaddr_in address;

    socklen_t len=sizeof(address);

    bzero( &address, sizeof( address ) );

    address.sin_family = AF_INET;

    inet_pton( AF_INET, ip, &address.sin_addr );

    address.sin_port = htons( port );

 

    int sock = socket( PF_INET, SOCK_STREAM, 0 );

    assert( sock >= 0 );

        int num;

        int ret=connect(sock,(struct sockaddr*)&address,len);

    if ( ret < 0 )

    {

        printf( "connect errno: %s\n", strerror(errno) );

    }

    else

    {

        while( (num=recv(sock,buf,sizeof(buf),0))>0 ){

            fprintf(fp,"%s",buf);

            memset(buf,'\0',sizeof(buf));

        }

        

        close( sock );

    }

 

    close( sock );

    return 0;

}
```

  

测试环境：linux Ubuntu 32位系统 CPU Intel i5-4258U @ 2.40GHz \*4 内存2G

![](https://img-blog.csdn.net/20160311173529786?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

  

根据以上对比，使用sendfile的系统负载要低一些，cpu使用率要低很多，整体速度和send基本差不多，估计是当今系统计算速度太快，看不出什么明显区别。不过对比web服务器的话区别还是很大的。