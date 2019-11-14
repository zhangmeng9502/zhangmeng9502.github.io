---
title: 初识prometheus监控系统
---

## prometheus是什么？

  Prometheus是一个由SoundCloud构建的开源监控告警系统,自2012年成立以来，许多公司和组织都采用了Prometheus.它现在是一个独立的开源项目，独立于任何公司维护。2016年Prometheus加入CNCF(Cloud Native Computing Foundation)，成为继Kubernetes之后的第二个托管项目。它的源码托管在github：https://github.com/prometheus/prometheus.git

## prometheus监控系统功能有哪些？

以下功能仅作为概括说明，具体实现将在架构模块分别体现。

  - 由metric和k/v组成的时序的多维数据模型。

  - 灵活的查询语句（PromQL)

  - 无依赖存储，每个prometheus是独立的。支持 local 和 remote 不同模型。

  - 采用 http 协议，使用 pull 模式拉取数据；并且支持push模型，通过
  intermediary gateway实现。

  - 监控目标可以采用服务发现或静态配置的方式。

  - 支持多种统计数据模型，图形化友好。

## prometheus 不适合做什么？

  Prometheus 不适合做审计计费，因为它的数据是按一定时间采集的，关注的更多是系统的运行瞬时状态以及趋势，即使有少量数据没有采集也能容忍，但是审计计费需要记录每个请求，并且数据长期存储，这个和 Prometheus 无法满足，可能需要采用专门的审计系统。

## prometheus的优势？

  prometheus 适合记录以及查询时间序列；通过服务发现或者静态配置去获取监控的 targets，使得组件间的耦合度更低， 更轻量级；提供了灵活而强大的查询语句PromQL; 架构更易伸缩等

## 架构

### 架构图如下：

{% img [class name] /pictures/Architecture.png %}

### prometheus监控系统各个模块是如何默契合作的？

  1. Prometheus server 定期从静态配置的 targets 或者服务发现的 targets 拉取数据。

  2. 当新拉取的数据大于配置内存缓存区的时候，Prometheus 会将数据持久化到磁盘（如果使用 remote storage 将持久化到云端）。

  3. Prometheus 可以配置 rules，然后定时查询数据，当条件触发的时候，会将 alert 推送到配置的 Alertmanager。

  4. Alertmanager 收到警告的时候，可以根据配置，聚合，去重，降噪，最后发送警告。

  5. 可以使用 API， Prometheus Console 或者 Grafana 查询和聚合数据。
