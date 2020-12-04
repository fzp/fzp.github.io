---
title: Prometheus 告警
date: 2019-11-19
tags:
  - alert
  - Prometheus
---

Prometheus如何处理警报？本文以例子为导向，深入浅出地介绍Prometheus的告警配置。

<!--more-->

## 简介

Prometheus服务器本身并不处理告警信。他将告警管理模块AlertManger单独了出来。

因此，Prometheus的告警分成两步，首先由Prometheus服务器定时根据告警规则进行计算。如果生成警报（akert）, 就推送到AlertManager处理。AlertManager会管理警报，将它们分组，去重，路由到对应的接收者等。

## Prometheus规则计算

在Prometheus的配置文件下添加用于警报的规则文件。

```yaml
rule_files:
  - '/etc/prometheus/alertmanager.rules'
```

`alertmanager.rules`是我们自己创建的文件。下面是官方文档的一个例子：

```yaml
groups:
- name: example
  rules:

  # Alert for any instance that is unreachable for >5 minutes.
  - alert: InstanceDown
    expr: up == 0
    for: 5m
    labels:
      severity: page
    annotations:
      summary: "Instance {{ $labels.instance }} down"
      description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 5 minutes."

  # Alert for any instance that has a median request latency >1s.
  - alert: APIHighRequestLatency
    expr: api_http_request_latencies_second{quantile="0.5"} > 1
    for: 10m
    annotations:
      summary: "High request latency on {{ $labels.instance }}"
      description: "{{ $labels.instance }} has a median request latency above 1s (current value: {{ $value }}s)"
```

alert是按照group来组织的，每一个group都由name,name后面跟着的rules,rules下是具体的alert. alert主要由下面各个部分组成：

表达式(expr)：Prometheus自己的PromQL表达式，比如`up == 0`是up这个metrics的值为0。

持续时间(for): PromQL表达式持续时间大于for指定的时间将会将会发送给alertManager.

标签(labels):给alert的标签，可用于后续alertManager的管理或发送给接收者。

注释（annotations):对alert的描述。

标签和注释不是一般的字符串,它是Go语言的模板语言，可以在字符串中添加`{{}}`插入变量。常见的变量有`$labels`，它是表达式中的标签。还有`$value`，它是表达式的值。

alert的生命周期分为3个状态:

1. inactive: expr为假
2. pending: expr为真但未达到for要求的时间
3. firing: expr为真且达到for要求的时间

## AlertManager警报处理

AlertManager是一个单独的模块，它有自己的配置文件。下面是一个官方例子。来自[这里](https://github.com/prometheus/alertmanager/blob/master/doc/examples/simple.yml)。

```yaml
global:
  # The smarthost and SMTP sender used for mail notifications.
  smtp_smarthost: 'localhost:25'
  smtp_from: 'alertmanager@example.org'
  smtp_auth_username: 'alertmanager'
  smtp_auth_password: 'password'
  # The auth token for Hipchat.
  hipchat_auth_token: '1234556789'
  # Alternative host for Hipchat.
  hipchat_api_url: 'https://hipchat.foobar.org/'

# The directory from which notification templates are read.
templates: 
- '/etc/alertmanager/template/*.tmpl'

# The root route on which each incoming alert enters.
route:
  # The labels by which incoming alerts are grouped together. For example,
  # multiple alerts coming in for cluster=A and alertname=LatencyHigh would
  # be batched into a single group.
  #
  # To aggregate by all possible labels use '...' as the sole label name.
  # This effectively disables aggregation entirely, passing through all
  # alerts as-is. This is unlikely to be what you want, unless you have
  # a very low alert volume or your upstream notification system performs
  # its own grouping. Example: group_by: [...]
  group_by: ['alertname', 'cluster', 'service']

  # When a new group of alerts is created by an incoming alert, wait at
  # least 'group_wait' to send the initial notification.
  # This way ensures that you get multiple alerts for the same group that start
  # firing shortly after another are batched together on the first 
  # notification.
  group_wait: 30s

  # When the first notification was sent, wait 'group_interval' to send a batch
  # of new alerts that started firing for that group.
  group_interval: 5m

  # If an alert has successfully been sent, wait 'repeat_interval' to
  # resend them.
  repeat_interval: 3h 

  # A default receiver
  receiver: team-X-mails

  # All the above attributes are inherited by all child routes and can 
  # overwritten on each.

  # The child route trees.
  routes:
  # This routes performs a regular expression match on alert labels to
  # catch alerts that are related to a list of services.
  - match_re:
      service: ^(foo1|foo2|baz)$
    receiver: team-X-mails
    # The service has a sub-route for critical alerts, any alerts
    # that do not match, i.e. severity != critical, fall-back to the
    # parent node and are sent to 'team-X-mails'
    routes:
    - match:
        severity: critical
      receiver: team-X-pager
  - match:
      service: files
    receiver: team-Y-mails

    routes:
    - match:
        severity: critical
      receiver: team-Y-pager

  # This route handles all alerts coming from a database service. If there's
  # no team to handle it, it defaults to the DB team.
  - match:
      service: database
    receiver: team-DB-pager
    # Also group alerts by affected database.
    group_by: [alertname, cluster, database]
    routes:
    - match:
        owner: team-X
      receiver: team-X-pager
      continue: true
    - match:
        owner: team-Y
      receiver: team-Y-pager


# Inhibition rules allow to mute a set of alerts given that another alert is
# firing.
# We use this to mute any warning-level notifications if the same alert is 
# already critical.
inhibit_rules:
- source_match:
    severity: 'critical'
  target_match:
    severity: 'warning'
  # Apply inhibition if the alertname is the same.
  equal: ['alertname', 'cluster', 'service']


receivers:
- name: 'team-X-mails'
  email_configs:
  - to: 'team-X+alerts@example.org'

- name: 'team-X-pager'
  email_configs:
  - to: 'team-X+alerts-critical@example.org'
  pagerduty_configs:
  - service_key: <team-X-key>

- name: 'team-Y-mails'
  email_configs:
  - to: 'team-Y+alerts@example.org'

- name: 'team-Y-pager'
  pagerduty_configs:
  - service_key: <team-Y-key>

- name: 'team-DB-pager'
  pagerduty_configs:
  - service_key: <team-DB-key>
  
- name: 'team-X-hipchat'
  hipchat_configs:
  - auth_token: <auth_token>
    room_id: 85
    message_format: html
    notify: true
```

通过上面的配置文件实例大概就可以了解了alertManager的功能和使用方法。

`global`是用于设置一些全局的默认配置。

`template`是用于设置发送发送给接受者的消息格式，它使用了Go语言的模板引擎。

`receiver`是alert消息的接收者，现在alertManger可以设置`webhook`,`e-mail`和`Slack`接收者。

`inhibit_rules`是一些静默规则，用于当某些alert生效时，把其他的alert静默掉。

`routes`是比较重要的配置， 它把特定的alert发送给指定的接收者。
它的数据结构是一棵树，当子节点的配置项会继承父节点的配置项。当alert发生时，它会从路由树的根节点进入，根节点必须是能够匹配所有alert。相当于是默认节点，保证至少有一个接收者会处理alert.接着它会遍历子节点，根据配置项`continue`的不同，它在发现了一个匹配项后，会立即返回(true)或者继续遍历(false).

它的匹配方法是根据metrics的标签进行精确匹配(match)或者正则匹配(match_re)。

另外，值得注意的是它消除重复alert的方式。为了避免同类型的错误重复告警，alertManager按组的方式管理alert的。分组的方式也是使用label,`group_by`配置项指定根据哪些label进行分组，如果这些label相同，那么alert就会分到同一组里面。

除了分组外，alertManager还通过限定同一组alert的发送间隔来避免重复告警。相关的配置项有3个，很容易混淆，所以我具体地接受一些：

`group_wait`:alert一般不是同时发生的，为了让符合条件的alert分到同一组，就需要让最先到的alert等待一段时间，这段时间就是`group_wait`,`group_wait`后才发送一组alert.

`group_interval`:再发送一组alert后，如果有新的alert进入到同一组中，新的alert就需要等待`group_interval`的时间。

`repeat_interval`:即使没有新的alert进来，旧的那组alert也会在`repeat_interval`后重新发送。

## 总结

Prometheus的alert管理并非十分复杂，通过它的配置文件我们就能基本了解它的功能和原理。但它有些配置项容易混淆，这些就需要仔细区分了。
