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

# 结论

```bash
grep Pss /proc/[1-9]*/smaps | awk '{total+=$2}; END {printf "%d kB\n", total }'
```

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



# 进程是如何申请内存的？

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/memory-apply.png)

看完这张图提问个问题？

当我的机器64位，不开启swap分区, 内存为2GB时，是否可以申请4GB的内存？

理论上最大能申请 128 TB 大小的虚拟内存， 因此即使物理内存只有2G, 也是可以申请4G内存的。

# 进程是如何回收内存的？

应用程序通过 malloc 函数申请内存的时候，实际上申请的是虚拟内存，此时并不会分配物理内存。

当应用程序读写了这块虚拟内存，CPU 就会去访问这个虚拟内存， 这时会发现这个虚拟内存没有映射到物理内存， CPU 就会产生**缺页中断**，进程会从用户态切换到内核态，并将缺页中断交给内核的 Page Fault Handler （缺页中断函数）处理。

缺页中断处理函数会看是否有空闲的物理内存，如果有，就直接分配物理内存，并建立虚拟内存与物理内存之间的映射关系。

如果没有空闲的物理内存，那么内核就会开始进行**回收内存**的工作，回收的方式主要是两种：直接内存回收和后台内存回收。

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/memory-free-v1.png)



## 单进程内存统计？

```bash
00400000-00401000 r--p 00000000 00:147 14824848                          /pause
Size:                  4 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Rss:                   4 kB
Pss:                   0 kB
Shared_Clean:          4 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:         0 kB
Referenced:            4 kB
Anonymous:             0 kB
LazyFree:              0 kB
AnonHugePages:         0 kB
ShmemPmdMapped:        0 kB
FilePmdMapped:         0 kB
Shared_Hugetlb:        0 kB
Private_Hugetlb:       0 kB
Swap:                  0 kB
SwapPss:               0 kB
Locked:                0 kB
THPeligible:    0
VmFlags: rd mr mw me sd
```

- VSS – Virtual Set Size 虚拟耗用内存（包含共享库占用的内存）
- RSS – Resident Set Size 实际使用物理内存（包含共享库占用的内存）
- PSS – Proportional Set Size 实际使用的物理内存（比例分配共享库占用的内存）
- USS – Unique Set Size 进程独自占用的物理内存（不包含共享库占用的内存）

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/p623795.png)https://elixir.bootlin.com/linux/v4.4/source/fs/proc/task_mmu.c#L440



# 问题

通过Pss 计算的全部进程的大小与free used 大小差距很大， 为什么？

```bash
used Used memory (calculated as total - free - buffers - cache)

cat /proc/meminfo | egrep "AnonPages|Mapped"
```



# 相关文献

[4.1 为什么要有虚拟内存？ | 小林coding](https://www.xiaolincoding.com/os/3_memory/vmem.html#%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)

[常用的内存查询命令/指标的含义是什么_应用实时监控服务(ARMS)-阿里云帮助中心](https://help.aliyun.com/zh/arms/application-monitoring/memory-metrics)

[The /proc Filesystem — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/proc.html?highlight=Pss)

[【Linux】/proc/meminfo 解析 - aalanwyr - 博客园](https://www.cnblogs.com/aalan/p/17026258.html)


