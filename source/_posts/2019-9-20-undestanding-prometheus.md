---
title: 理解Prometheus
date: 2019-09-20
tags:
  - Monitoring
  - 监控
  - Prometheus
---

多维度时间序列是理解Promethues的关键

<!--more-->

## 简介

[Prometheus](https://prometheus.io/) 是一款监控软件。真正学习和使用过Promethues后，发现它对于监控业务的理解极其透彻。理解Promethues更像是在了解监控的本质。

## 监控的目的

为什么需要监控？监控的目的当然是获取服务运行时的状态。利用运行时的状态，我们可以做分析和告警的工作。

1. 故障分析定位
2. 告警
3. 趋势分析
4. 比较分析
5. ...

## 数据模型

我们要监控的对象是什么？

我们要监控的数据是怎么样的？要实现监控的目的，它们具有什么样的特征呢？

主要有两点：

1. 时间序列
2. 多维度

时间序列容易理解，监控一个最重要的目的是保证服务在任何时间点都能正常运作，所以必须是持续不断地监控服务状态。获取的数据必然形成一条时间序列。

多维度是什么意思？以监控一个集群中机器节点的CPU usage监控为例。
不同机器上的CPU usage的监控数据形成一条时间序列。它们组合在一起形成了一组时间序列。
每个节点有不同的IP, 我们可以用IP区分不同机器上CPU usage形成的时间序列。那么IP就是这组时间序列的一个维度。
同时不同节点可能使用不同的操作系统，那么操作系统就是另一个的维度。
维度是监控数据的关注点。通过对不同维度数据的聚合和分析，我们可以获得服务状态的重要信息。
从这个角度看，时间也是其中一个维度。只不过在监控中，时间维度是最为重要的。

总的来说，时间序列是监控数据的基本形式，多维度则是聚合时间序列的关键。

Prometheus的数据模型就是多维度的时间序列。

对应上述概念到Prometheus中，时间序列的每一个点称之为样本(sample)。每个样本有3个部分：

1. 指标（metric）
2. 时间戳（timestamp）
3. 值（value）

指标是需要被监控的对象，表示为指标名和标签的组合。格式如下：

```
<metric name>{<label name>=<label value>, ...}
```

标签代表一个维度，表示为key-value形式。指标名在Prometheus底层也是一个标签，他的标签名为__name__, 值就是指标名本身。

比如上面例子的一个指标可以写成:

```
cpu_usage{os='windows', ip='127,0,0,1'}
```

## 数据查询

在设计数据模型的时候，我们已经从分考虑了数据的聚合。因此监控系统能够高效地数据进行统计分析。Prometheus提供了强大的PromQL查询语言获取聚合数据。

依然从两个方面分析PromQL: 时间序列和多维度。

## 时间序列

### 瞬时向量

在上一节，我们已经了解了如何表示指标。指标在PromQL里面用于表示瞬时向量（Instant vector）。
瞬时向量的官方定义是一组时间序列，每条时间序列只包含一个值，且它们的时间戳是相同的。因为只是一个时间戳的值。所以是**瞬时**的。又由于是一组时间序列，所以形成了**向量**。

### 区间向量

在时间序列中，有时我们需要分析一个区间内的数据。这时候就需要区间向量（Range vector）。区间向量就是在原有的瞬时向量的基础上加入时间选择器`[]`。比如查找5分钟内的平均CPU利用率：

```
CPU_usage{}[5m]
```

### 偏移

另外无论是瞬时向量和区间向量，都是以当前（最新）时间为基准的。有时候我们希望访问旧的数据，需要移动我们的基准，使用offset关键字。

```
CPU_usage{}[5m] offset 5m
```

### 标量(Scalar)和字符串(String)

在PromQL中，还允许标量和字符串，比如10，“hello world”。
标量可以用于运算和可视化。例如和瞬时向量可以和标量进行加减乘除运算。只有一个时间序列的瞬时变量可以通过函数`scalar()`转化为标量。有时候在监控中，我们只需要知道当前的系统是不是健康的，当前磁盘空间是多少，只是后就可以用`scalar()`得到标量并在可视化工具中展示。

字符串在PromQL中还未被使用。

### 函数

PromQL还提供一些函数对瞬时向量和区间向量进行处理。下面简单介绍几个函数。

`scalar(v instance-vector)`是我们第一个见过的函数， 他将瞬时向量转化成标量。

`abs(v instant-vector)`可以将瞬时变量中的值转变为绝对值。

`increase(v range-vector)` 求区间向量的增长量（最新值减最旧值）并返回一个瞬时向量。

## 多维度

我们已经讨论过如何对单一的时间序列进行查询和计算。但很多时间有用的数据是通过聚合得到的。所以我们需要充分利用多维度的特性，对数据进行聚合。

### 聚合操作

根据Prometheus的数据模型，监控对象往往是以瞬时向量的形式给出的。瞬时向量是一组多维度的样本。这就要求PromQL能够在不同的维度对数据进行聚合。

PromQL提供了一组内置的聚合操作符作用于单一瞬时变量。如：

* sum (求和)
* min (最小值)
* max (最大值)
* avg (平均值)
* stddev (标准差)
* stdvar (标准差异)
* count (计数)
* count_values (对value进行计数)
* bottomk (后n条时序)
* topk (前n条时序)
* quantile (分布统计)

最后四个操作符支持参数(parameter)。

聚合语法如下：

```
<aggr-op>([parameter,] <vector expression>) [without|by (<label list>)]
```

without用于排除结果中标签对应的维度。by用于保留结果中标签对应的维度。默认情况下排除所有标签对应的维度。

例如计算HTTP请求总量的表达式：

```
sum(http_requests_total)
```

在application维度上对HTTP请求总量聚合的表达式:

```
sum(http_requests_total) by (application)
```

如果`http_requests_total`只有`application`, `instance`和`group`三个标签，则上面的表达式等价于：

```
sum(http_requests_total) without (instance, group)
```

### 瞬时变量间的运算

先来个例子感受下瞬时变量间的运算。

输入：

```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
method_code:http_errors:rate5m{method="put", code="501"}  3
method_code:http_errors:rate5m{method="post", code="500"} 6
method_code:http_errors:rate5m{method="post", code="404"} 21

method:http_requests:rate5m{method="get"}  600
method:http_requests:rate5m{method="del"}  34
method:http_requests:rate5m{method="post"} 120
```

PromQL表达式：

```
method_code:http_errors:rate5m{code="500"} / ignoring(code) method:http_requests:rate5m
```

PromQL表达式求的是在过去5分钟内，HTTP请求状态码为500的在所有请求中的比例。

结果：

```
{method="get"}  0.04            //  24 / 600
{method="post"} 0.05            //   6 / 120
```

瞬时向量与标量之间的运算时将瞬时向量的被一个样本和标量进行运算。那么瞬时向量和瞬时向量间是多个样本和多个样本间的结合。多个样本和多个样本又是如何匹配呢？是笛卡尔积吗？并没有必要，因为很多匹配是没有意义的。有意义的匹配是标签相同的样本。

瞬时向量间运算的基本匹配规则是找到完全匹配所有标签的两个时间序列进行运算。
在开头的例子中，我们会发现`ignoring()`关键字。它表示在匹配过程中，忽略某些标签。这是为了在运算符左右表达式不一致的依然能够匹配。类似地。我们有`on()`关键字。它与`ignoring()`相反，只匹配指定的标签。

重现审视例子中的表达式。

符合`method_code:http_errors:rate5m{code="500"}`要求的有两个样本`method_code:http_errors:rate5m{method="get", code="500"} `和`method_code:http_errors:rate5m{method="post", code="500"}`。

在匹配过程中`ignoring(code)`要求忽略code标签的影响。所以与两个样本匹配的右表达式样本为`method:http_requests:rate5m{method="get"}`和`method:http_requests:rate5m{method="post"}`。

最终样本值执行除法得到结果。

上面的例子是左右表达式一对一的结果。而事实上我们可能有一对多或者多对一的要求。比如：

```
method_code:http_errors:rate5m{method="get", code="500"}  24
method_code:http_errors:rate5m{method="get", code="404"}  30
```

对应

```
method:http_requests:rate5m{method="get"}  600
```

用于求出不同状态码在所有请求中的比例。这时候可以使用group_left或者group_right表达式。这种用法相对高级，具体的细节请参考官方的文档。

## 总结

数据模型和查询功能是Prometheus的核心。我们从监控的目标开始，分析了Prometheus的数据模型，最后介绍了PromQL查询语言，描绘了一条清晰的设计路线。可以看到，多维度时间序列是理解Promethues的关键所在。

PromQL的大致的样貌已经在前面展示出来了。但实际上PromQL比前面说的还要强大，比如支持布尔运算和集合运算，指标类型(metric types)等等。有兴趣的读者可以自行查阅官方的文档。另外，作为一个成熟的软件，Prometheus还有用于收集数据、服务发现、可视化以及告警的组件。这些留待日后有空慢慢分析。

## 参考
1. https://prometheus.io/docs/introduction/overview/
2. https://yunlzheng.gitbook.io/prometheus-book/
