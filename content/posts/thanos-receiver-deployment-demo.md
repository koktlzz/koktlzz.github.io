---
title: "Thanos Receiver"
date: 2022-01-18T09:51:42+01:00
draft: true
tags: ["Prometheus","Thanos"]
summary: " ..."
---

## Sidecar or Receiver

Sidecar 的缺点：通常每隔 2 小时才会将 Block 上传到对象存储中，因此 Querier 查询近期数据时需要向所有 Sidecar 发起请求。

Receiver 只推荐用于：多租户、一些 SaaS 环境（Openshift Operator 管理的 Prometheus）

Receiver 的缺点：自身的资源使用量较高，很可能 OOM（Receiver 端会保存和 Prometheus 一样的 TSDB 以及 WAL）。根据[官方文档](https://prometheus.io/docs/practices/remote_write/#memory-usage)，Prometheus 添加 Remote Write 后，需要将 WAL 拷贝到远端，会增加约 25% 的内存使用量。

采用 Receiver 模式的理由：

- 多租户
- 集群规模并不大，series 数量？sample per second ？
- Openshift Operator 管理的 Prometheus，IaaS/DBaaS 自建的非容器 Prometheus，不方便配置 Sidecar

## 快速开始

## 验证

## Future Work

此架构中 Ruler 的配置问题

Receiver、Compactor 和 Store 的 Limit 和 Request

Receiver 的 REPLICATION_FACTOR

Remote Write 的性能调优

多集群部署

## 参考文献

[Thanos Remote Write](https://thanos.io/v0.11/201812_thanos-remote-receive.md/)

[Achieve Multi-tenancy in Monitoring with Prometheus & Thanos Receiver](https://www.infracloud.io/blogs/multi-tenancy-monitoring-thanos-receiver/)

[Adopting Thanos at Lastpass](https://krisztianfekete.org/adopting-thanos-at-lastpass/)
