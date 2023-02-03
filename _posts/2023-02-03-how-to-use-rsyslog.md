---

title: "rsyslog文档"

categories:

- Blog

tags:

- Linux

---



# 简介

[syslog](https://juejin.cn/post/6858168312388386824)

**Syslog 是生成、转发和收集在 Linux 实例上生成日志的标准。Syslog 定义了严重性级别和设施级别，有助于用户更好地理解其计算机上生成的日志。**

[linux - What is the difference between syslog, rsyslog and syslog-ng? - Server Fault](https://serverfault.com/questions/692309/what-is-the-difference-between-syslog-rsyslog-and-syslog-ng)

[官方文档](https://www.rsyslog.com/)

特点：

- 多线程
- TCP、SSL、TLS、RELP
- MySQL、PostgreSQL、Oracle 等
- 过滤系统日志消息的任何部分
- 完全可配置的输出格式
- 适用于企业级中继链

# 安装

[Install](https://www.rsyslog.com/doc/v8-stable/installation/index.html)

```shell
sudo yum install rsyslog
sudo apt-get install rsyslog
sudo apk add rsyslog
```

不幸的是，发行版通常提供相当旧的 rsyslog 版本，因此您可能想要更新的东西。为了轻松做到这一点，我们提供了[包](https://www.rsyslog.com/doc/v8-stable/installation/packages.html)和[Docker 容器](https://www.rsyslog.com/doc/v8-stable/installation/rsyslog_docker.html)。

# 配置

**Rsyslogd 通过 rsyslog.conf 文件进行配置，该文件**通常位于`/etc`. 默认情况下，rsyslogd 读取文件`/etc/rsyslog.conf`. 这可以通过命令行选项更改。

请注意，**可以**通过在线[rsyslog 配置构建器](http://www.rsyslog.com/rsyslog-configuration-builder/)工具以交互方式构建配置。

## 基本结构

[basic_struct](https://www.rsyslog.com/doc/v8-stable/configuration/basic_structure.html)

- 所有规则**总是**被完全评估，不管过滤器是否匹配（所以我们**不会**在第一次匹配时停止）。如果要停止消息处理，则必须明确执行“丢弃”操作（由波浪字符或停止命令表示）。如果执行丢弃，消息处理立即停止，而不评估任何进一步的规则。

## Filter

[filters](https://www.rsyslog.com/doc/v8-stable/configuration/filters.html)

## RainerScript

https://www.rsyslog.com/doc/v8-stable/rainerscript/index.html

流程控制, 变量

## Actions

[Actions](https://www.rsyslog.com/doc/v8-stable/configuration/actions.html)

Action 对象描述要对消息执行的操作。它们通过[输出模块](https://www.rsyslog.com/doc/v8-stable/configuration/modules/idx_output.html)实现。

动作对象有不同的参数：

- 适用于所有动作且特定于动作的那些。这些记录在下面。

- 动作队列的参数。虽然它们也适用于所有参数，但它们是特定于队列的，而不是特定于操作的（例如，它们与规则集中使用的相同）。[这些在队列参数](https://www.rsyslog.com/doc/v8-stable/rainerscript/queue_parameters.html)下单独记录。

- 特定于动作的参数。这些特定于某种类型的操作。它们由相关[输出模块](https://www.rsyslog.com/doc/v8-stable/configuration/modules/idx_output.html)记录。

## Input

[Input](https://www.rsyslog.com/doc/v8-stable/configuration/input.html)

`input`顾名思义，该对象描述消息输入源。没有输入，根本不会发生任何处理，因为没有消息进入 rsyslog 系统。输入是通过[输入模块](https://www.rsyslog.com/doc/v8-stable/configuration/modules/idx_input.html)实现的。

输入对象有不同的参数：

- 适用于所有输入并且通常可用于所有输入的那些。这些记录在下面。

- 输入特定参数。这些特定于某种类型的输入。它们由相关[输入模块](https://www.rsyslog.com/doc/v8-stable/configuration/modules/idx_input.html)记录。

## Output

[output](https://www.rsyslog.com/doc/v8-stable/configuration/output_channels.html)
