## 什么是promQL
PromQL是Prometheus监控系统的查询语言。它是 Prometheus 的一项核心功能，可以对收集的**时间序列数据**进行仪表板、警报和查询，所有这些都基于同一系统中的一种统一语言。PromQL 允许您选择、聚合和运算时间序列。


## 数据模型

### 时间序列

![Alt text](image.png)

prometheus 的[数据模型](https://prometheus.io/docs/concepts/data_model/)

### 指标类型

prometheus 允许exporter使用四种不同的[指标类型](https://prometheus.io/docs/concepts/metric_types/)。

- Counter 只增不减，如：http请求数，cpu使用时间
- Gauge 可增可减，如：温度，内存使用量，磁盘使用量
- Histogram 对观察对象（通常是持续时间、响应大小）进行采样，通过桶计数的方式进行统计
- Summary 与Histogram类似，不同点是它计算滑动时间窗口上的可配置分位数
## promQL语言

PromQL是一种嵌套函数语言，用来分析时间序列数据的一种查询语言。我们可以先看下promQL查询的结构和类型是怎么样，以及随着时间的推移如何计算时序值。
我们将了解promQL表达式的结构、表达式结果类型、表达式节点类型、查询类型和间隔。
### 嵌套结构

PromQL 是一种嵌套函数式语言。这意味着您将要查找的数据描述为一组嵌套的表达式，每个表达式都作为中间值参与计算（无副作用）。每个中间值都用作其周围表达式的参数或操作数，而查询的最外层表达式表示在表、图形或类似用例中看到的最终返回值。

```
# Root of the query, final result, approximates a quantile.
histogram_quantile(
  # 1st argument to histogram_quantile(), the target quantile.
  0.9,
  # 2nd argument to histogram_quantile(), an aggregated histogram.
  sum by(le, method, path) (
    # Argument to sum(), the per-second increase of a histogram over 5m.
    rate(
      # Argument to rate(), the raw histogram series over the last 5m.
      demo_api_request_duration_seconds_bucket{job="demo"}[5m]
    )
  )
)
```
![Alt text](image-1.png)
PromQL 表达式不仅是整个查询，而且是查询的任何嵌套部分（如上面的部分 rate(…) ），每个嵌套的查询都可以单独运行。在上面的示例中，每个注释行表示一个表达式。

### 表达式结果类型
- 字符串 一个简单的字符串值
- 标量 一个简单的浮点值
- 即时向量 一组时间序列，每个时间序列包含一个样本，所有时间序列共享相同的时间戳
- 范围向量 一组时间序列，其中包含每个时间序列随时间变化的一系列数据点

### 表达式节点类型
我们如何写promQL表达式呢，表达式节点有下面10种类型

- aggregation 聚合(sum,min,max,avg,stddev,stdvar,count,group,count_values,bottomk,topk,quantile),聚合表达式中参数是即时向量,结果也只能是即时向量 如 sum without (instance) (http_requests_total)
- binaryExpr 二元运算符组合成多元表达式（+-*/%^==!=...） 如 http_requests_total - http_requests_error_total
- call 函数表达式（abs,absent,absent_over_time...）,参数是标量或即时向量、范围向量，返回结果目前是即时向量、标量 如 abs(http_requests_total)
- matrixSelector 范围向量选择器 如 http_requests_total{job="job"}[1m]
- vectorSelector 向量选择器 如 http_requests_total{job="job"}
- subquery 子查询 支持指定查询范围和精度 返回范围向量 如 rate(http_requests_total[1m])[30m:1m]
- numberLiteral 数字 如 1、1e-1
- stringLiteral 字符串 如 "version"
- parenExpr 元括号 (up) 
- unaryExpr 一元表达式 -1^2
- placeholder 

### 查询类型和间隔
#### 即时查询
参数：
- promQL表达式
- time

即时查询主要用于检测和表格，即时查询可以返回任何有效的 PromQL 表达式类型（字符串、标量、即时和范围向量）。
![Alt text](image-2.png)

#### 范围查询
范围查询主要用于图表，显示给定时间范围内的 PromQL 表达式。范围查询的工作方式与许多完全独立的即时查询完全相同，这些即时查询在给定时间范围内的后续时间步长进行评估。当然，这是在引擎盖下高度优化的，在这种情况下，Prometheus实际上并没有运行许多独立的即时查询。范围查询允许传入即时向量类型或标量类型表达式，但始终返回范围向量

参数：
- promQL表达式
- start 
- end
- step
![Alt text](image-3.png)
## 查询演示

### 执行过程分析

### 表达式演示