---
toc: true
title: "Alertmanager参数"
categories:
  - devops
tags:
  - linux
---

# 背景

  (php服务)线上服务器收到大量爬虫请求，导致php-fpm进程频繁OOM, 临时使用nginx 对此类ip进行封禁。后续通过分析，发现php-fpm设置不合理，需要最合理调整，因此需要获取正常情况下一个php-fpm进程占用了多少内存？ 

# 如何查看进程内存？

方式1： ps

```shell
> ps -p 2902371 -o pid,rss,cmd
    PID   RSS CMD
2902371 70864 php-fpm: pool www
```

方式2： top

```shell
> top -p 2902371
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
2902371 www-data  20   0  891876  70864  47656 S   0.0  0.1   0:01.45 php-fpm
```

方式3：

```shell
> cat /proc/2902371/status
VmRSS:       70864 kB
```

# RSS 是什么？

resident set size, the non-swapped physical memory that a task has used (in kilobytes).
Resident Set Size（常驻集大小）是指一个任务已使用的非交换物理内存大小（以千字节为单位）。




