---
title: "Thanos Receiver"
date: 2022-01-18T09:51:42+01:00
draft: true
tags: ["Prometheus","Thanos"]
summary: " ..."
---

## 前言

Sidecar 的缺点：通常每隔 2 小时才会将 Block 上传到对象存储中，因此 Querier 查询近期数据时需要向所有 Sidecar 发起请求。

## 参考文献

[Thanos Remote Write](https://thanos.io/v0.11/201812_thanos-remote-receive.md/)

[Achieve Multi-tenancy in Monitoring with Prometheus & Thanos Receiver](https://www.infracloud.io/blogs/multi-tenancy-monitoring-thanos-receiver/)

[Adopting Thanos at Lastpass](https://krisztianfekete.org/adopting-thanos-at-lastpass/)
