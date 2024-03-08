---
toc: true
title: "如何使用lxd?"
categories:
  - devops
tags:
  - lxd
  - docker
---

# 前言

此文档大部分基于：

入门文档: https://ubuntu.com/blog/the-lxd-2-0-story-prologue 翻译而来

官方文档: https://linuxcontainers.org/lxd/docs/master/



如果你了解kvm/docker相关技术，lxd学习便事半功倍

因为lxd也是一种容器,所以可以和docker进行比较学习，即docker的功能点在lxd中如何实现

# 简介

*什么是LXD*?

最简单的说，LXD是一个守护程序，它提供一个REST API来驱动LXC容器。 其主要目标是提供与虚拟机相似的用户体验，但使用Linux容器而不是硬件虚拟化。

*LXD与Docker / Rkt有何关系*？

LXD与Docker / Rkt有何关系？ 这是到目前为止我们得到的最多的问题，所以让我们立即解决它！ LXD专注于系统容器，也称为机器容器。也就是说，LXD容器运行完整的Linux系统，就像在金属或VM中运行时一样。 这些容器通常会运行很长时间，并且基于干净的分发映像。传统的配置管理工具和部署工具可以与LXD容器一起使用，就像您将其用于VM，云实例或物理机一样。 相比之下，Docker专注于短暂的，无状态的，最小的容器，这些容器通常不会进行升级或重新配置，而只会被完全替换。这使得Docker和类似项目比机器管理工具更接近于软件分发机制。 两种模式也不是互斥的。您绝对可以使用LXD向您的用户提供完整的Linux系统，然后用户可以在其LXD容器中安装Docker来运行他们想要的软件。

*为什么选择LXD？*

Canonical的团队已经从事LXC多年了。 LXC擅长于其工作，也就是说，它提供了非常好的一组低级工具以及一个用于创建和管理容器的库。 但是，此类低级工具不一定对用户友好。 他们需要大量的初步知识，以了解他们的工作方式和工作方式。 保持与较旧容器和部署方法的向后兼容性还阻止了LXC默认使用某些安全功能，从而为用户提供了更多手动配置。 我们认为LXD是解决这些缺点的机会。 作为一个长期运行的守护程序，它使我们可以解决很多LXC限制，例如动态资源限制，容器迁移和有效的实时迁移，它还使我们有机会提出一种新的默认体验，该体验默认情况下是安全的，并且在很多情况下 更加以用户为中心。

# 虚拟化技术

Lxd/kvm/docker



容器隔离技术

- namespace # 进程隔离（待学习）
- cgroups # 资源隔离(待学习)



(个人理解, 正确性待参考)

LXD 功能是提供一个完整的linux 操作系统, 这与KVM有着相似的能力，但不同的是KVM需要使用qemu做硬件软件两个层次的虚拟化,从而实现与宿主机的资源隔离。LXD则是采用与docker相似的容器隔离技术, 不需要硬件层次的虚拟化，拥有一个rootfs即可实现实例的创建

lxd设计的目的是交付一个完整的linux操作系统

docker设计的目的是交付一个应用程序

进入容器可见区别：

对于lxd而言. PID=1的进程是/sbin/init, 这个主进程是不会发生变化的，同时他可以拉起某些系统进程，类似systemd等

对于docker而言，pid=1的进程取决于你的cmd或者entrypoint命令，例如sshd， 他不会去启动某些系统进程，这就产生一种现象,即docker 会自杀，根本原因在于主进程pid=1的进程结束任务，主进程退出，容器自动exit

# LXD的主要组成部分

构成LXD的组件有很多，这些组件通常在LXD目录结构，其命令行客户端以及API结构本身中可见。

## 容器

- 文件系统:rootfs
- 配置选项列表
- 设备例如磁盘，字符/块Unix设备和网络接口
- 容器从中继承配置的一组配置文件
- 一些属性（容器架构，临时或持久名称）
- 某些运行时状态（将CRIU用于检查点/还原时）

支持配置项（常见）

```shell
boot.autostart        # 自动重启
environment.*         # 为容器添加key:value的环境变量
limits.cpu            # 限制cpu的个数
limits.cpu.allowance  # 限制cpu的使用率
limits.kernel.*       # 内核资源限制
limits.memory         # 内存限制
limits.processes      # 进程个数限制
security.privileged   # 提权操作
```

--project 3.0.3版本暂未使用， 与其对应的RBAC暂未使用

## 快照

容器快照与容器相同，只是它们是不可变的，它们可以重命名，销毁或还原，但不能以任何方式进行修改。 值得注意的是，由于我们允许存储容器运行时状态，因此有效地为我们提供了“有状态”快照的概念。 也就是说，具有在快照时回滚容器（包括其CPU和内存状态）的能力。

## 镜像

LXD是基于镜像的,所有的LXD容器都来自镜像，镜像通常是干净的。

可以发布一个容器，从中制作镜像，然后供本地或者远程LXD主机使用

名称采用sha256哈希值， 使用方式类似于docker

“ ubuntu：”提供稳定的Ubuntu映像

“ ubunt-daily：”提供Ubuntu的每日构建

“图像：”是社区运行的图像服务器，它使用上游LXC模板为许多其他Linux发行版提供图像

远程映像由LXD守护程序自动缓存，并且自它们在过期之前最后一次使用以来一直保留了几天（默认为10天）。 此外，LXD还会自动更新远程映像（除非另有说明），以使映像的最新版本始终在本地可用。

## 配置

配置文件是一种在一个位置定义容器配置和容器设备，然后将其应用于任意数量的容器的方法。 一个容器可以应用多个配置文件。在构建最终容器配置（称为扩展配置）时，将按照定义它们的顺序应用配置文件，当找到相同的配置密钥或设备时，这些配置文件将相互覆盖。然后，在此之上应用本地容器配置，覆盖配置文件中的所有内容。 LXD附带了两个预配置的配置文件： 除非用户提供了其他配置文件列表，否则“默认”将自动应用于所有容器。该配置文件当前仅做一件事，为容器定义“ eth0”网络设备。 “ docker”是可以应用于要允许其运行Docker容器的容器的配置文件。它要求LXD加载一些必需的内核模块，打开容器嵌套并设置一些设备条目。

## 远程控制

如前所述，LXD是一个网络守护程序。 因此，它附带的命令行客户端可以与多个远程LXD服务器以及图像服务器进行通信。 默认情况下，我们的命令行客户端附带以下已定义的遥控器 本地：（默认远程，通过unix套接字与本地LXD守护程序对话） ubuntu ：（提供稳定版本的Ubuntu图像服务器） ubuntu-daily ：（提供每日构建的Ubuntu图像服务器） 图像：（[images.linuxcontainers.org](http://images.linuxcontainers.org/)图像服务器） 这些遥控器的任何组合都可以与命令行客户端一起使用。 您还可以添加任意数量的配置为侦听网络的远程LXD主机。 如果它们是公共映像服务器，则为匿名方式；或者在管理远程容器时，通过身份验证后，则匿名方式。 正是这种远程机制使与远程图像服务器进行交互以及在主机之间复制或移动容器成为可能。

## 安全

LXD通过使用LXC库使用的主要安全功能是：

**kernel namespaces**: 尤其是用户名称空间，它是使容器所做的一切与系统其余部分分开的一种方式。 LXD默认使用用户名称空间（与LXC相反），并允许用户在绝对需要时基于每个容器将其关闭（将容器标记为“特权”）。

**Seccomp**：筛选一些潜在危险的系统调用。

**AppArmor**：对安装，套接字，ptrace和文件访问提供其他限制。专门限制跨容器通信。

**Capabilities**: 为了防止容器加载内核模块，改变主机系统时间，…

**CGroups**：限制资源使用并防止对主机的DoS攻击

## REST API

LXD所做的一切都是通过其REST API完成的。 客户端和守护程序之间没有其他通信通道。 可以通过本地unix套接字访问REST API，只需要使用组成员身份进行身份验证即可，也可以使用客户端证书通过HTTPs套接字进行身份验证。 REST API的结构与上述不同组件匹配，使用起来非常简单直观。 当需要更复杂的通信机制时，LXD将协商Websocket，并将其用于其余的通信。 这用于交互式控制台会话，容器迁移和事件通知。 LXD 2.0附带了/1.0稳定的API。 我们不会破坏/1.0 API端点内的向后兼容性，但是我们可能会向它添加额外的功能，这将通过声明客户端可以寻找的其他API扩展来发出信号。

## 扩展性

尽管LXD提供了一个很好的命令行客户端，但是该客户端并不能在多个主机上管理数千个容器。 对于这样的用例，我们有nova-lxd，这是一个OpenStack插件，它使OpenStack对待LXD容器的方式与对待VM的方式完全相同。 使用OpenStack API来管理网络，存储和负载平衡，这允许在大量主机上进行大型LXD部署。

# 安装配置

*在哪里获得LXD以及如何安装 *

有很多方法可以获取最新最好的LXD。 我们建议您将LXD与最新的LXC和Linux内核一起使用，以从其所有功能中受益，但我们会尝试在适当的情况下进行降级以支持较早的Linux发行版。

```shell
sudo apt install lxd
sudo lxd init
```

注意:

```shell
ExecStartPre=/usr/lib/x86_64-linux-gnu/lxc/lxc-apparmor-load (code=exited, status=0/SUCCESS)

```

场景：在lxd中运行lxd， lxd init执行失败, 请手动跳过apparmor 安全检查（可联系运维管理员）

注意：

lxd启动成功依然需要执行lxd init进行初始化操作，根据需要回车(默认就好)

## 命令

```shell
# 容器相关
lxc launch ubuntu: aliasName #创建并启动容器
lxc launch images:d2f4d8cc3f1a apline-test # 根据镜像id常见并启动
lxc init ubuntu: # 创建但不启动容器
lxc list # 获取所有容器信息
lxc list --fast# 输出部分信息
lxc info apline-test # 获取容器相关信息
lxc stop apline-test # 停止容器
lxc stop <container> --force # 强制停止
lxc start apline-test # 启动容器
lxc restart apline-test # 重启容器
lxc restart <container> --force # 强制重启
lxc pause apline-test # 暂停容器
lxc delete apline-test # 删除容器
lxc delete apline-test --force # 强制性删除容器

# 配置文件相关
lxc profile list # 可用的配置文件
lxc profile show default # 展示配置文件详细信息
lxc profile edit default # 编辑配置文件
lxc profile apply <container> <profile1>,<profile2>,<profile3>,... # 指定某个容器应用哪些配置项
lxc config edit apline-test # 编辑某个容器的配置项
lxc config show apline-test # 显示某个容器的配置项
lxc config show --expanded apline-test # 显示容器配置的详细信息

# 进入容器
lxc exec <container> bash # 进入容器
lxc exec apline-test sh # apline 没有bash 

# 文件传输
lxc file pull immense-condor/root/1.txt . # 从容器中拷贝文件
lxc file pull immense-condor/etc/hosts - # 从容器中读取显示到终端
lxc file push <source> <container>/<path> # 将本地文件发送到容器中
lxc file edit <container>/<path> # 编辑容器中的文件

# 快照管理
lxc snapshot <container> # 为某个容器创建快照
lxc snapshot <container> <snapshot name> # 创建并命名
lxc restore <container> <snapshot name> # 恢复快照
lxc move <container>/<snapshot name> <container>/<new snapshot name> # 为某个快照重命名
lxc copy immense-condor/snap-xsw xsw-u # 从快照中启动一个新的容器
lxc delete <container>/<snapshot name> # 删除快照

# 复制
lxc copy <source container> <destination container> 

# 移动
lxc move <old name> <new name>

lxc image list ubuntu: # 获取ubuntu相关的所有镜像
lxc image list images: # 获取可用的全部镜像

# 端口映射
lxc config device add ubuntu-test sshport22331 proxy listen=tcp:0.0.0.0:22331 connect=tcp:127.0.0.1:22
```

# 资源控制

- 内存配额
- CPU限制
- IO优先级
- 带宽
- 磁盘使用限制

## disk

这也许是最需要和最明显的。 只需在容器的文件系统上设置大小限制，然后对容器执行该限制即可。

这正是LXD可以让您做到的！ 不幸的是，这远比听起来复杂得多。 Linux没有基于路径的配额，相反，大多数文件系统仅具有用户和组配额，这些配额对容器几乎没有用处。

这意味着，如果您使用的是ZFS或btrfs存储后端，那么LXD现在仅支持磁盘限制。 可能也可以为LVM实现此功能，但这取决于与其一起使用的文件系统，并且在与实时更新结合使用时会变得棘手，因为并非所有文件系统都允许在线增长，而几乎所有文件系统都不允许在线收缩。

## CPU

- **Just give me X CPUs**
- **Give me a specific set of CPUs (say, core 1, 3 and 5)**
- **Give me 20% of whatever you have**
- 每测量200毫秒，给我50毫秒（仅此而已)

```shell
lxc config xsw-u limits.cpu 1
```



## Memory

支持配置x MB的RAM

同样也支持百分比的请求（10%）

```shell
 lxc config set xsw-u limits.memory 256MB
```

# 镜像管理

LXD中具有高级图像缓存和预加载支持，以使图像存储保持最新。 区别于docker

```shell
# 获取镜像
lxc image copy ubuntu:14.04 local: # 下载镜像到本地,不启动容器
lxc image copy ubuntu:12.04 local: --alias old-ubuntu # 为镜像起别名
lxc image copy ubuntu:15.10 local: --copy-aliases # 获取远程镜像的别名
lxc image copy images:gentoo/current/amd64 local: --alias gentoo --auto-update # 自动更新镜像
lxc image import <tarball> # 从tar包导入镜像
lxc image import <tarball> --alias random-image # 从tar包导入镜像，并起别名
lxc image import https://dl.stgraber.org/lxd --alias busybox-amd64 # 从url中导入镜像

# 管理本地镜像
lxc image list # 列出所有本地镜像
lxc image list amd64 # 根据别名过滤镜像
lxc image list os=ubuntu # 根据键值对形式过滤镜像
lxc image info ubuntu # 查看镜像的详细信息
lxc image edit <alias or fingerprint> # 编辑某个镜像文件
lxc image delete <alias or fingerprint> # 删除某个镜像文件
lxc image export old-ubuntu . # 导出镜像成tar文件

# 根据容器创建镜像
lxc launch ubuntu:14.04 my-container
lxc exec my-container bash
<do whatever change you want>
lxc publish my-container --alias my-new-image
lxc publish my-container/some-snapshot --alias some-image

```

# 远程协议

可实现在本地操作远程lxd

# LXD运行docker

为了使所有这些都能正常工作，有很多有趣的方面，我们在Ubuntu 16.04中包含了所有这些内容： 具有CGroup名称空间支持的内核（4.4 Ubuntu或4.6 mainline） 使用LXC 2.0和LXCFS 2.0的LXD 2.0 Docker的自定义版本（或使用我们提交的所有补丁构建的版本） 一个由用户名称空间限制时行为的Docker映像，或者将父LXD容器设置为特权容器（security.privileged = true）

docker 的profile需要自己定义

```
lxc profile create docker
lxc profile edit docker

config:

  linux.kernel_modules: overlay, nf_nat

  security.nesting: "true"

description: Profile supporting docker in containers

devices:

  aadisable:

    path: /sys/module/apparmor/parameters/enabled

    source: /dev/null

    type: disk

name: docker
```



```shell
lxc launch ubuntu-daily:16.04 docker -p default -p docker # 运行一个容器
lxc config set docker security.privileged true # 提权操作，非必须生产环境下不允许添加
lxc restart docker
```

## 未完结