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

router:
  # How long to initially wait to send a notification for a group
  # of alerts. Allows to wait for an inhibiting alert to arrive or collect
  # more initial alerts for the same group. (Usually ~0s to few minutes.)
  [ group_wait: <duration> | default = 30s ]

  # How long to wait before sending a notification about new alerts that
  # are added to a group of alerts for which an initial notification has
  # already been sent. (Usually ~5m or more.)
  [ group_interval: <duration> | default = 5m ]

  # How long to wait before sending a notification again if it has already
  # been sent successfully for an alert. (Usually ~3h or more).
  # Note that this parameter is implicitly bound by Alertmanager's
  # `--data.retention` configuration flag. Notifications will be resent after either
  # repeat_interval or the data retention period have passed, whichever
  # occurs first. `repeat_interval` should not be less than `group_interval`.
  [ repeat_interval: <duration> | default = 4h ]
```

## resolve_timeout

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

## group_wait

# 注意

## duration 类型值

[类型要求](https://prometheus.io/docs/alerting/latest/configuration/#duration)
[校验规则](https://github.com/prometheus/common/blob/main/model/time.go#L204)
