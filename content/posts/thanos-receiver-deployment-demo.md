---
title: "使用 Thanos 实现多集群（租户）的统一监控"
date: 2022-01-18T09:51:42+01:00
draft: true
tags: ["Prometheus","Thanos"]
summary: " ..."
---

## 前言

[Thanos](https://github.com/thanos-io/thanos) 已成为目前 Kubernetes 集群监控的标准解决方案之一。它基于 Prometheus 之上，可以为我们提供：

- 全局的指标查询视图
- 几乎无限的数据保留期限
- 包含 Prometheus 在内所有组件的高可用性

在拟定监控方案之前，阅读一些成熟的 [用户案例](https://thanos.io/tip/thanos/getting-started.md/#blog-posts) 是十分必要的。这些博文先分析了各自团队的集群现状以及当前监控方案难以解决的痛点，再对目前流行的几种技术栈进行对比，最后介绍投入生产使用的部署方案，因此非常值得一读。

不过，由于 Thanos 的组件众多，且每种组件都有较多参数需要配置。对于刚接触 Thanos 的用户来说，可能难以快速上手。考虑到上述博文均未给出组件的具体配置信息，而官方提供的 [部署清单](https://github.com/thanos-io/kube-thanos) 又稍显复杂并缺少详细说明。因此本文将介绍一个使用 Thanos 实现多集群（租户）监控的简单 Demo，希望能对试图尝鲜 Thanos 的用户有所帮助。它实现了以下功能：

- 能够监控多个集群，并提供一个全局的指标查询视图；
- 每个集群都对应了一个唯一的租户名称，可以通过租户标签区分不同集群的指标数据；
- 如果想要监控更多的集群，只需在新集群中部署 Prometheus 并配置远程写入；
- 简单的告警规则和告警消息推送。

## Sidecar vs. Receiver

Thanos 支持 [Sidecar](https://thanos.io/tip/components/sidecar.md/) 和 [Receiver](https://thanos.io/tip/components/receive.md/) 两种部署模式。它们各有利弊，需要我们根据实际情况进行取舍。

Sidecar 通常每隔 2 小时才会把 Prometheus 采集到的指标上传到对象存储中，因此 Query 查询近期数据时需要向所有 Sidecar 发起请求并合并返回结果。但这并非是 Thanos 团队引入 Receiver 的决定性因素。

> Receiver is only recommended for uses for whom pushing is the only viable solution, for example, analytics use cases or cases where the data ingestion must be client initiated, such as software as a service type environments.

实际上，Receiver 只推荐用于多租户以及 Prometheus 配置受限的场景下，比如：

- 被监控集群使用某些 SaaS 服务，如 Openshift Operator 部署的 Prometheus；
- 由于安全或权限问题，监控团队无法在被监控集群中配置 Sidecar；
- 被监控集群使用非容器化部署的 Prometheus。

这是因为 Receiver 会暂存多个 Prometheus 实例的 TSDB，当数据量较大时可能会发生 OOM。另外根据 [官方文档](https://prometheus.io/docs/practices/remote_write/#memory-usage)，开启远程写入还将增加 Prometheus 约 25% 的内存使用。

我们并非一定要在 Receiver 和 Sidecar 之间做出抉择，比如 [Lastpass](https://krisztianfekete.org/adopting-thanos-at-lastpass/) 就采用了 Sidecar 与 Receiver 混合部署的方式：

![9dad5f8992fcb42354b85d46c276afa](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/9dad5f8992fcb42354b85d46c276afa.png)

由于我们的 Demo 优先考虑架构的简洁性和易用性以及对多租户的支持，对资源占用和性能的要求并不高，因此将选用 Receiver 模式。

## 快速开始

- Kubernetes 版本: v1.16.9-aliyun.1
- 环境：阿里云 ACK

```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: thanos
EOF

git clone https://github.com/koktlzz/thanos-k8s-deployment.git
cd thanos-k8s-deployment
kubectl apply -k overlays/aliclound/
```

## 架构说明

## 验证

### 使用 Operator 部署 Prometheus

```shell
git clone https://github.com/coreos/kube-prometheus.git
cd kube-prometheus
kubectl create -f manifests/setup

# wait for namespaces and CRDs to become available, then
kubectl create -f manifests/
```

> 在国内环境拉取  k8s.gcr.io 上的镜像可能会失败，需要将镜像名称改为 bitnami/kube-state-metrics:2.3.0 和 willdockerhub/prometheus-adapter:v0.9.0。

### 为 Prometheus 实例配置远程写入

远程写入 `kubectl edit -n monitoring prometheus k8s`：

```yaml
spec:
  remoteWrite:
    # local cluster
    - url: http://thanos-receiver.thanos.svc.cluster.local:19291/api/v1/receive
    # cluster kazusa
    - url: http://nginx-proxy-kazusa.uuid.cn-shanghai.alicontainer.com/api/v1/receive
    # cluster setsuna
    - url: http://nginx-proxy-setsuna.uuid.cn-shanghai.alicontainer.com/api/v1/receive
```

随后将在 Query UI 中看到三个集群（租户）的实例：

![20220121161530](https://cdn.jsdelivr.net/gh/koktlzz/NoteImg@main/20220121161530.png)

## Future Work

本文只是介绍了一个最简化的配置，如果要投入到生产环境，还需要：

此架构中 Ruler 的配置问题

Receiver、Compactor 和 Store 的 Limit 和 Request

Receiver、Compactor 和 Store 的磁盘

Receiver 的 REPLICATION_FACTOR

Remote Write 的性能调优 [Remote write tuning | Prometheus](https://prometheus.io/docs/practices/remote_write/)

receiver 证书 tls

多集群部署

## 参考文献

[Multi cluster monitoring with Thanos · Banzai Cloud](https://banzaicloud.com/blog/multi-cluster-monitoring/)

[Thanos Remote Write](https://thanos.io/v0.11/201812_thanos-remote-receive.md/)

[Achieve Multi-tenancy in Monitoring with Prometheus & Thanos Receiver](https://www.infracloud.io/blogs/multi-tenancy-monitoring-thanos-receiver/)

[Adopting Thanos at Lastpass](https://krisztianfekete.org/adopting-thanos-at-lastpass/)