---
title: prometheus-server
author: MinGuk
---

## 前言

   本篇文章将简要介绍prometheus-server相关概念以及如何定制他的配置文件(包括prometheus.yml 和 alert rules)

## prometheus 架构
{% img [class name] /pictures/Architecture.png %}

## prometheus-server

这里需要把prometheus-server 与prometheus监控系统做区分.从上面的架构图可以看出prometheus-server是整个prometheus生态系统的核心, 它是一个时序数据库, 这里需要读者补充一个知识点:什么是时序数据库？

1. 数据模型

    `prometheus`一条`metric`由一个度量名称和一组标签组成, 并且由一个度量名称和一组标签唯一标识.例如:

    `<metric name>{ <label name>=<label value>, ...}`

2. metrics数据类型:

 - Counter:counter是一个累计度量指标, 它是一个只能递增的数值.计数器主要用于统计服务的请求数、任务完成数和错误出现的次数等等.

 - Gauge:Gauge是一个度量指标, 它表示一个既可以递增, 又可以递减的值.

 - Histogram:柱状图.一个度量类型为histogram的metric在采集数据期间会自动生成三个metric.

   例如:`<metric name>`为`<basename>`的Histogram, 在采集数据期间会生成:

       <basename>_bucket: 分别统计该指标各个bucket的累计值
       <basename>_sum: 统计<basename>value的累积和
       <basename>_count: 统计<basename>的个数

 - Summary: summary是分位图统计, (通常的使用场景:请求持续时间和响应大小).

3. Prometheus提供了PromQL用于查询数据

    举个例子, prometheus job/exporter的状态可以通过指标:up 来查询
    先来一个直观的, 下图是prometheus自带的UI, query=up时prometheus返回所有job/exporter的状态, value=0表明该job/exporter状态是down, value=1表明该job/exporter正常工作

    {% img [class name] /pictures/query_up.png %}

## PromQL

prometheus 提供了PromQL 用于实时查询和聚合时间序列, 我们可以通过

Prometheus API查询到已经在prometheus已监控到的数据.API使用指南可以看这里: https://prometheus.io/docs/prometheus/latest/querying/api/#expression-queries

我们可以根据上文所说的通过prometheus UI调用prometheus api 或者直接调用.

这里给出一个简单的API调用举例:

http请求:http://127.0.0.1:9091/api/v1/query?query=up

返回值:

```
{
    "status":"success", 
    "data":{
        "resultType":"vector", 
        "result":[
            {
                "metric":{
                    "__name__":"up", 
                    "instance":"10.0.0.5:9091", 
                    "job":"prometheus"
                }, 
                "value":[
                    1554290431.799, 
                    "1"
                ]
            }, 
            {
                "metric":{
                    "__name__":"up", 
                    "instance":"10.0.0.5:9198", 
                    "job":"openstack_exporter"
                }, 
                "value":[
                    1554290431.799, 
                    "0"
                ]
            }
        ]
    }
}
```

## prometheus-server 配置文件

prometheus.yml

```
global:
  scrape_interval: 15s  # prometheus从tartget收集数据的时间间隔
  evaluation_interval: 30s # alert rules评估时间间隔
  scrape_timeout: 10s # prometheus 从target收集数据的超时时间
rule_files: # 告警规则文件的路径
# target配置
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 180s # 覆盖global中定义的scrape_interval
    scrape_timeout: 10s # 覆盖global中定义的scrape_timeout
    static_configs:
      # target url
      - targets:
          - 127.0.0.1:9090
# alertmanager配置
alerting:
  alertmanagers:
    - scheme: http
      static_configs:
        # alertmanager url
        - targets:
          - 127.0.0.1:9093
```
这是一个很简单的配置, 其他字段详细解释请参考:

https://prometheus.io/docs/prometheus/latest/configuration/configuration/

## reload prometheus配置文件

prometheus允许在运行过程中reload配置文件, reload有两种方式:
1. 向Prometheus进程发送SIGHUP 请求

```
kill -HUP pid
```

2. 向prometheus发送HTTP POST请求`

```
curl -X POST http://localhost:9090/-/reload --web.enable-lifecycle
```

## prometheus rules
 prometheus支持两种类型的rules:recording rules和alerting rules.

### recording rules

recording rules 支持我们预先计算经常需要使用或者计算时资源消耗较大的表达式, 并将其结果保存为一组新的时间序列.这样查询预先计算的结果会比每次执行原始表达式快得多.这对于dashboard尤其有用, dashboard需要在每次刷新时重复查询相同的表达式, 查询原始表达式会造成短时间内后端接收的请求很多, 而查询预先计算的结果会大大缩短执行时间, 也可以减轻后端的压力.

recording_example.rules
```
# 每一个告警规则都存在groups中
groups:
  - name: example_1 # 组名, 在一个规则文件中必须是唯一的
    rules:
    # record 与 expr 对应, 将expr表达式的查询结果存储在record中
    # record:要输出的时间序列的名称
    # expr: 查询表达式
    # labels: 在存储结果前可以添加或覆盖一些标签
      - record: job:http_inprogress_requests:sum
        expr: sum(http_inprogress_requests) by (job)
        labels:
          [ <labelname>: <labelvalue> ]

  - name: example_2
    rules:
      - record: job:http_inprogress_requests:total
        expr: sum(http_inprogress_requests)
```

### alerting rules

prometheus会定期评估alerting rules中的表达式是否成立.prometheus.yml的配置项evaluation_interval为评估时间间隔, 即每隔多长时间评估一次.

alerting rules 示例

alerting_example.rules
```
groups:
  - name: ovs rules # 组名, 在文件中必须唯一
    rules:
    - alert: OVS Port is down # 告警名称
      expr: ovs_port_up == 0 # Prom表达式
      for: 1m
      labels: # 在告警之前添加或覆盖的标签
        severity: Critical # 这里添加了告警级别的标签, 也可以添加其他标签
      annotations: # 添加到告警的注释, 可以添加多个, 支持自定义.
        description: "OpenVSwitch Port {{ $labels.name }} on host {{ $labels.hostname }} is down"
```

注:
1. for子句是可选的, 表达式成立时, 产生的告警并不立即进入firing状态, 而是处于pending状态, 当持续1m后表达式仍然成立, 此时该告警从pending状态转换为firing状态；当没有for字段时, 表达式一旦成立, 立即产生firing状态的告警.
2. labels子句指定要附加到警报的一组附加标签, 现有的标签都将被覆盖, 标签值可以模板化
3. annotations 子句指定了一组用于存储较长信息的额外信息标签.如添加告警描述, 处理建议等, annotations的值可以模版化
4. 模版化:$labels变量保存告警实例的标签键/值对, $value保存告警实例的评估值.
5. prometheus alerting rules解决了告警生成的问题, 但它并不是一个完全成熟的通知告警方案.告警产生后, 还需要对告警进行分组, 静默等操作, 这一功能由prometheus生态系统中的alertmanager来实现, prometheus只负责周期性地向alertmanager发送原始的告警数据.

在浏览器输入:http://*.*.*.*:9090/alerts 可以看到哪些警告处于active(pending or firing)状态.如下图

{% img [class name] /pictures/alerting.png %}

## prometheus语法检查工具

prometheus提供了告警规则语法的检查工具: promtool, 下面给出几个常用命令

通用格式: `promtool [<flags>] <command> [<args> ...]`

```
Commands:

- help [<command>...]: Show help.

- check config <config-files>...: Check if the config files are valid or not.

- check rules <rule-files>...: Check if the rule files are valid or not.

- test rules <test-rule-file>...: Unit tests for rules.

- query instant <server> <expr>: Run instant query.
```

promtool示例

```
/prometheus $ promtool check config /etc/prometheus/prometheus.yml
Checking /etc/prometheus/prometheus.yml
  SUCCESS: 1 rule files found

Checking /etc/prometheus/rules/up.yml
  SUCCESS: 1 rules found

/prometheus $ promtool query instant http://10.0.0.8:9090 up
up{instance="10.0.0.8:9090", job="prometheus"} => 1 @[1575882449.642]
up{instance="10.0.0.8:9100", job="node_exporter"} => 1 @[1575882449.642]
```
