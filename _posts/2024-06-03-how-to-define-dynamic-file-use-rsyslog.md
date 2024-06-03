---
toc: true
title: "使用Rsyslog动态生成日志文件"
categories:
  - devops
tags:
  - rsyslog
---

如果对rsyslog 不熟悉, 请移步[rsyslog](https://www.rsyslog.com/doc/)

# 需求

使用rsyslog 将服务日志流转到线下。配置文件中存在大量重复配置

```bash
:syslogtag, startswith, "chd-user" -/var/log/production/chd-user-prd/tomcat.log;t_msg
:syslogtag, startswith, "chd-view" -/var/log/production/chd-view-prd/tomcat.log;t_msg
:syslogtag, startswith, "chd-xiaoe" -/var/log/production/chd-xiaoe-prd/tomcat.log;t_msg
```

思考：

是否可用简化配置？即使用模版动态生成文件

通过分析发现 文件的输出位置 都是以syslogtag 区分的，例如：

- chd-user 输出到 /var/log/production/chd-user-prd 
- chd-view 输出到  /var/log/production/chd-view-prd

可通过配置模版，生成动态的文件 

# Template

https://www.rsyslog.com/doc/configuration/templates.html

```bash
# name 定义模版名称, type 此模版类型
template(name="DynamicFile" type="list") {
  # 定义常量
	constant(value="/var/log/production/")
	# 定义变量，从rsyslog属性中获取， submatch=0 为全部
  property(name="programname" regex.expression="(.*-prod).*" regex.type="ERE" regex.submatch="1")
  # 定义常量
  constant(value=".log")
}
```

此模版会根据`programname` 生成一个动态的 `字符串`。

# Action

```bash
# startswith 过滤
if ($syslogtag startswith "chd") then {
    action(type="omfile" dynaFile="DynamicFile")
    stop
}
# 正则过滤
if (re_match($syslogtag, "chd.*|chd1.*")) then {
    action(type="omfile" dynaFile="DynamicFile")
    stop
}
```

