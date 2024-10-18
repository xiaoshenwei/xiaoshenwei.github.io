---
toc: true
title: "基于Ansible的发布系统设计"
categories:
  - devops
tags:
  - ansible
---

# 基于Ansible的发布系统设计

## 场景

对于公司内部一些旧服务，直接采用tomcat 或者jar 包部署，暂无法docker 化。

因此当前内部 服务的部署类型 分为以下三类

- 宿主机部署，将构建产物发送到一个或者多个目标机器启动
- docker 部署，将构建产物打包成镜像，在目标机器上启动
- k8s, 基于docker 镜像，将服务部署在k8s 集群中

对于k8s 本身提供了各种api 可以供外部调用，因此 发布系统 主要考虑 其他两种类型即可。



## 选型

ansible 无agent 的方式, 目标机器无需太多变动。

## 架构图

![](https://raw.githubusercontent.com/xiaoshenwei/xiaoshenwei.github.io/master/assets/images/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_d3181a1e-76fb-401b-b3db-922d6f04a401.png)

## 功能设计

### 支持指定节点发布

app: 部署在192.168.200.100， 192.168.200.101， 192.168.200.102， 192.168.200.103 四个节点上， 某些场景下用户只希望更新100， 101 两个节点

全量发布:

```yaml
- hosts: 192.168.200.100,192.168.200.101,192.168.200.102,192.168.200.103
  tasks:
    - name: test connection
      ping:
```

指定节点发布：

```yaml
- hosts: 192.168.200.100,192.168.200.101
  tasks:
    - name: test connection
      ping:
```

要支持此能力， 发布系统需要 **动态** 生成一份配置文件， ansible 启动前加载此配置文件。

例如: app.json

```json
{
  "targets_hosts": 
  	[
      "192.168.200.100",
      "192.168.200.101",
      "192.168.200.102",
      "192.168.200.103"
    ]
}
```

```yaml
- name: 加载本地配置文件
  hosts: localhost
  tasks：
  - name: Init Config From Json File"
    include_vars:
      file: "app.json"
      name: ctx
  # 根据 配置文件中的字段 设置一个 动态host
  - name: Add remote targets based on local configuration
    add_host:
      name: "{{ item }}"
      groups: dynamic_targets
    loop: "{{ ctx.target_hosts }}"

- name: 开始发布
  hosts: dynamic_targets
  tasks:
    - name: test connection
      ping:
```

### 支持多vpc

不同vpc 需要不同参数例如：

- ssh key
- ssh port
- ......

创建hosts 文件

```ini
[test]
192.168.200.100
192.168.200.101
192.168.200.102
192.168.200.103

[aliyun]
10.16.2.1
10.16.2.2

[tx]
172.16.1.1
```

创建 groups_vars 目录, 为某个组设置环境变量

```ini
# test
ansible_ssh_private_key_file: /home/deploy/.ssh/sf
```

```ini
# aliyun
ansible_ssh_private_key_file: /home/deploy/.ssh/java
ansible_ssh_executable: ssh
```

服务只关心  target_hosts IP 列表 ansible 自动解析连接参数， 从而实现跨vpc。

### 为远程主机设置参数

当应用发布 特定版本 需要执行与版本相关的参数时:

例如： app-b02.tar `b02`为版本号， 当此版本发布时， 需要在目标主机解压缩， 因为版本号由发布系统指定，ansible 处理此任务不能把 命令写死: tar -xvf app-$VERSION.tar, 因此`VERSION` 应该传递给远程主机。

app.json

```json
{
  "name": "app",
  "version": "b02",
  "targets_hosts": 
  	[
      "192.168.200.100",
      "192.168.200.101",
      "192.168.200.102",
      "192.168.200.103"
    ]
}
```

```yaml
- name: 加载本地配置文件
  hosts: localhost
  tasks：
  - name: Init Config From Json File"
    include_vars:
      file: "app.json"
      name: ctx
  # 根据 配置文件中的字段 设置 运行时参数
  - name: Set Runtime Global Vars
    set_fact:
      runtime_vars:
        APPNAME: "{{ ctx.name }}"
        VERSION: "{{ ctx.version }}"
	- name: Echo
	  shell: "echo $VERSION; echo $APPNAME"
	  environment: "{{ runtime_vars }}"
```

### 实现优雅关机

优雅关机 一般 需要支持两种

- http request
- command

例如： app.json

```json
{
  "name": "app",
  "version": "b02",
  "targets_hosts": [],
  "graceful_shutdown": {
    "url": "http://localhost:8080/shutdown",
    "method": "POST",
    "timeout": 10,
    "command": "docker stop app",
    "pause": 10
  }
}
```

```yaml
---
# tasks file for graceful_shutdown
- name: Graceful Shutdown With HTTP
  uri:
    url: "{{ ctx.graceful_shutdown.url }}"
    method: "{{ ctx.graceful_shutdown.method | default('GET') }}"
    timeout: "{{ ctx.graceful_shutdown.timeout | default(10) }}"
  ignore_errors: yes
  when: ctx.graceful_shutdown.url is not none and ctx.graceful_shutdown.url | length > 0

- name: Graceful Shutdown With Command
  shell: "{{ ctx.graceful_shutdown.command }}"
  ignore_errors: yes
  when: ctx.graceful_shutdown.command is not none and ctx.graceful_shutdown.command | length > 0

- name: Wait for the specified delay
  pause:
    seconds: "{{ ctx.graceful_shutdown.pause_seconds | default(10) }}"
  when: (ctx.graceful_shutdown.pause_seconds | default(10)) > 0

```

### 进行健康检查

1. 请求接口，等待200响应码，支持重试，超时
2. 检查响应内容，是否符合预期

```json
# app.json
{
  "name": "app",
  "version": "b02",
  "targets_hosts": [],
  "health_check": {
    "url": "http://localhost:8080/healthz",
    "method": "GET",
    "timeout": 10,
    "retries": 3,
    "delay": 5,
    "except": "ok"
  }
}
```

```yaml
# tasks file for health_check
- name: HealthCheck wait for 200 status code
  uri:
    url: "{{ ctx.health_check.url }}"
    method: "{{ ctx.health_check.method | default('GET') }}"
    timeout: "{{ ctx.health_check.timeout | default(10) }}"
  register: result
  retries: "{{ ctx.health_check.retries | default(60) }}"
  delay: "{{ ctx.health_check.delay | default(5) }}"
  until: result.status == 200
  when: ctx.health_check.url is not none and ctx.health_check.url | length > 0

- name: HealthCheck match except content
  uri:
    url: "{{ ctx.health_check.url }}"
    method: "{{ ctx.health_check.method | default('GET') }}"
    timeout: "{{ ctx.health_check.timeout | default(10) }}"
    return_content: true
  register: result
  failed_when: result.content != ctx.health_check.except
  when: 
    - ctx.health_check.url
    - ctx.health_check.url | length > 0
    - ctx.health_check.except is not none
    - ctx.health_check.except | length > 0

```

### docker 服务发布流程

- 获取目标容器
- stop 目标容器
- rm 目标容器
- 启动新容器

```json
# app.json
{
  "name": "app",
  "version": "b02",
  "targets_hosts": [],
  "docker_container": {
    "name": "app",
    "image": "xxxxx",
    ... # 定义可能需要的参数
  }
}
```

```yaml
- name: Info Container
  community.docker.docker_container_info:
    name: "{{ ctx.name }}"
  register: result

- name: Docker Stop Container
  community.docker.docker_container:
    name: "{{ ctx.name }}"
    state: stopped
    stop_timeout: "{{ ctx.docker_container.stop_timeout | default(10) }}"
  when: result.exists

- name: Docker Remove Container
  community.docker.docker_container:
    name: "{{ ctx.name }}"
    state: absent
  when: result.exists

- name: Docker Run Container
  community.docker.docker_container:
    name: "{{ ctx.name }}"
    image: "{{ ctx.docker_container.image }}"
    network_mode: "{{ ctx.docker_container.network_mode | default('host') }}"
    restart_policy: "always"
    ports: "{{ ctx.docker_container.ports | default([]) }}"
    volumes: "{{ ctx.docker_container.volumes | default([]) }}"
    command: "{{ ctx.docker_container.command | default('')  }}"
    
```

### 非docker 服务发布

- 下载目标文件
- 停止旧服务
- 启动新服务

```jsob
# app.json
{
  "name": "app",
  "version": "b02",
  "targets_hosts": [],
  "artifact": "http://download.host/app-b01.tar",
  "lifecycle": {
  	"pre_start": ["tar -xvf app-b01.tar"],
  	"start": ["java -jar app.jar"],
  	"post_start": [],
  	"restart": ["java -jar app.jar"]
  }
}
```



### 制品仓库选择

对于docker 服务上传镜像到harbor 。

对于文件类型，使用nexus 作为制品库 存储构建产物。

### tsocks 使用

1. 搭建ssh 隧道 到socks 代理
2. tsocks 包装 ssh 命令

```shell
# 搭建隧道
ssh -o ServerAliveInterval=60 -D 1234 -Nf deploy@socks_proxy

# 创建tssh 
vim /usr/bin/tssh

# 使用 libtsocks 透明地通过 SOCKS 代理运行 ssh 命令
tsocks ssh "$@"

# 设置 ansible
ansible_ssh_executable: tssh
```

socks 代理安全加固

socks 代理服务器 设置 authorized_keys

```
restrict,port-forwarding,permitopen="*:22",permitopen="*:7249",permitopen="*:80",command="true" ssh-rsa xxx user@host
```

