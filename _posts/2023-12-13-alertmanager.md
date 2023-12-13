---
toc: true
title: "Alertmanager参数"
categories:
  - devops
tags:
  - prometheus
  - alertmanager
---

# 背景

生产中使用 [alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) 进行告警信息的接受，alertmanager 的配置比较难以理解，因此结合实际使用中碰到的一些问题来分析下各参数的用途

# 参数

```yaml
global:
  # ResolveTimeout is the default value used by alertmanager if the alert does
  # not include EndsAt, after this time passes it can declare the alert as resolved if it has not been updated.
  # This has no impact on alerts from Prometheus, as they always include EndsAt.
  [ resolve_timeout: <duration> | default = 5m ]
```

## ResolveTimeout

问题:

如果你配置了除prometheus 以外的其他告警源
{: .notice--warning}
当某些告警还在触发时, alertmanager 可能将其自动Resolved.

原因：

resolve_timeout 默认值是5m，超过5m 会自动判断此告警已恢复

解决方式：

如果想要禁用 AlertManager 自动 Resolved 的功能:
```yaml
resolve_timeout: 1d
```


  
