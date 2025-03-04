---
title: Nginx 性能优化配置
tags:
  - permanent-note
  - nginx
date: 2025-03-04
time: 10:24
aliases:
---
# 1 配置文件句柄

执行 `ulimit -a` 查看系统文件句柄信息

```shell
[root@VM-4-7-centos ~]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 14690
max locked memory       (kbytes, -l) unlimited
max memory size         (kbytes, -m) unlimited
open files                      (-n) 100001
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 8192
cpu time               (seconds, -t) unlimited
max user processes              (-u) 14690
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```

`open files` 为当前最大可打开文件句柄数，可以通过命令 `ulimit -n [number]` 设置，但仅限于当前用户，永久修改需要修改 `/etc/security/limits.conf` ，更多请参考 [ulimit 命令](ulimit%20命令.md)。

# 2 TCP 连接配置

文件 `/etc/sysctl.conf` 中添加配置

```properties
# It enables fast recycling of TIME_WAIT sokcets. Known to cause some issues with hoststated(NAT and load balancing) if enabled, should be used with caution.
net.ipv4.tcp_tw_recycle = 0
# This allows resuing sockets in TIME_WAIT state for new connections when it is safe from protocol viewpoint. It is generally a safer alternative to tcp_tw_recycle
net.ipv4.tcp_tw_reuse = 1
# Determines the time that must elapse before TCP/IP can release a closed connection and reuse its resource. During this TIME_WAIT state, reopening the connection to
# the client costs less then establishing a new connection. By reducing the value of the entry, TCP/IP can release closed connections faster, making more resources
# available for new connections.
net.ipv4.tcp_fin_timeout = 5
# The wait time between isAlive interval probes(seconds)
net.ipv4.tcp_keepalive_intvl = 15
# The number of probes before timing out.
net.ipv4.tcp_keepalive_probes = 3
# The time of keepalive remined time(seconds)
net.ipv4.tcp_keepalive_time = 300

# The length of the syn quene
net.ipv4.tcp_max_syn_backlog = 65535
# The length of the tcp accept queue
net.core.somaxconn = 65535

# port range
net.ipv4.ip_local_port_range = 1024 65000
```

应用生效：`sysctl -p`

# 3 Nginx 配置

## 3.1 Worker 多线程

```nginx
user                    nginx;
worker_processes        auto;
worker_cpu_affinity     auto;
worker_rlimit_nofile    409600;
error_log               /var/log/nginx/error.log warn;
pid                     /var/run/nginx.pid;
```

## 3.2 主配置

```nginx
events {
    use                 epoll;
    worker_connections  409600;
    multi_accept        on;
    accept_mutex        off;
}

http {
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    log_format  main    '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

    access_log          /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;

    keepalive_timeout   300;
    keepalive_requests  20000000;

    open_file_cache max=10240 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 1;

    gzip                on;
    gzip_min_length     1k;
    gzip_buffers        4 16k;
    gzip_http_version   1.0;
    gzip_comp_level     2;
    gzip_types          text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary           on;

    include /etc/nginx/conf.d/*.conf;
}
```

events表示启用epoll模型来处理高并发的请求，epoll模型是linux底层的高性能、高并发处理模型，nginx也是基于此实现的。 在http的主配置中，开启`sendfile`、`tcp_nopush`和`tcp_nodelay`； 同时keepalive一定要配置上，timeout表示超时时间；requests表示同一个连接接收多少请求后断开； 剩下的就是一些常用的配置，记得在最后如果有其他额外的配置，要放到`conf.d`目录下。

## 3.3 负载节点配置

```nginx
upstream nis {
    server obp2:5001 max_fails=0;
    server obp2:5002 max_fails=0;
    server obp2:5003 max_fails=0;
    server obp2:5004 max_fails=0;
    keepalive 300;
}
```

配置为长连接。

注意`max_fails=0`表示忽略当前负载节点的报错，如果不配置，默认值为1，即当负载节点报错1次后，即暂停向当前节点分发请求。

## 3.4 监听端口信息

```nginx
server {
    listen              443 ssl http2 backlog=65535;
    listen              [::]:443 ssl http2 backlog=65535;
    server_name         localhost.com;
    access_log          off;

    ssl_certificate     your-cert-file-path;
    ssl_certificate_key your-cert-key-file-path;
    ssl_session_timeout 1d;
    ssl_session_cache   shared:SSL:50m;
    ssl_session_tickets off;

    # modern configuration
    ssl_protocols       TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    location /nis {
        proxy_pass http://nis;
        proxy_set_header Host $host:$server_port;
        proxy_set_header X-Forwarded-Host $server_name;
        proxy_set_header proxy_x_forward_ip $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```


监听项后的backlog参数决定了tcp连接队列的最大长度，默认nginx的tcp监听连接队列是511。

配置设置 HTTP 版本为 1.1，默认开启长连接。

```nginx
proxy_http_version 1.1;
proxy_set_header Connection "";
```

## 3.5 配置文件模板

```nginx
user                    nginx;
worker_processes        auto;
worker_cpu_affinity     auto;
worker_rlimit_nofile    409600;
error_log               /var/log/nginx/error.log warn;
pid                     /var/run/nginx.pid;

events {
    use                 epoll;
    worker_connections  409600;
    multi_accept        on;
    accept_mutex        off;
}

http {
    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    log_format  main    '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

    access_log          /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;

    keepalive_timeout   300;
    keepalive_requests  20000000;

    open_file_cache max=10240 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 1;

    gzip                on;
    gzip_min_length     1k;
    gzip_buffers        4 16k;
    gzip_http_version   1.0;
    gzip_comp_level     2;
    gzip_types          text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary           on;

    upstream nis {
        server obp2:5001 max_fails=0;
        server obp2:5002 max_fails=0;
        server obp2:5003 max_fails=0;
        server obp2:5004 max_fails=0;
        keepalive 300;
    }

    server {
        listen              443 ssl http2 backlog=65535;
        listen              [::]:443 ssl http2 backlog=65535;
        server_name         localhost.com;
        access_log          off;

        ssl_certificate     your-cert-file-path;
        ssl_certificate_key your-cert-key-file-path;
        ssl_session_timeout 1d;
        ssl_session_cache   shared:SSL:50m;
        ssl_session_tickets off;

        # modern configuration
        ssl_protocols       TLSv1.2;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
        ssl_prefer_server_ciphers on;

        # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
        add_header Strict-Transport-Security max-age=15768000;

        location /nis {
            proxy_pass http://nis;
            proxy_set_header Host $host:$server_port;
            proxy_set_header X-Forwarded-Host $server_name;
            proxy_set_header proxy_x_forward_ip $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Port $server_port;
            proxy_http_version 1.1;
            proxy_set_header Connect "";
        }
    }

    include /etc/nginx/conf.d/*.conf;
}
```

# 4 Reference
* [Nginx高并发配置 \| LotsOfNotes](https://jon-xia.gitbook.io/workspace/linux-fu-wu-qi-4/nginx-gao-bing-fa-pei-zhi)
* [【Nginx】Nginx高并发业务场景下的Linux系统配置及Nginx配置 \| KaKa's blog](https://lipeng1667.github.io/2019/08/05/high-concurrent-system-configration-with-nginx-and-rhel/)
* [Nginx极简实战—Nginx服务器高性能优化配置，轻松实现10万并发访问量-阿里云开发者社区](Nginx极简实战—Nginx服务器高性能优化配置，轻松实现10万并发访问量-阿里云开发者社区.md)
* [ChatGPT - Nginx 高性能配置](https://chatgpt.com/share/67c18b67-d7f4-8009-a636-bcb46d509343)