---
title: k8s 每日一问 - preStop、滚动更新、 流量丢失问题
date: '2023-02-14 10:24:39'
sidebar: true
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


>  preStop 执行的时间，它和 livenessprobe 的影响关系


1. 在preStop执行之前会停止（StopLivenessAndStartup）健康检查探针

2. 当gracePeriod小于2时，会设置最小值为2 (minimumGracePeriodInSeconds),总是给容器一个最小的关闭窗口，以避免不必要的sigkill

3. 在 preStop 结束之前不会发送 TERM信号

**[为后端服务配置合理的preStop Hook](https://help.aliyun.com/document_detail/398740.html#12)**

在后端服务滚动更新时，Nginx ingress Controller会将正在终止的Pod的端点摘除，同时保持还在处理的请求的连接。如果后端服务Pod在收到结束信号后立即退出，就可能会导致正在处理的请求失败，或者由于时序问题，部分流量仍被转发到已经退出的Pod中，导致部分流量产生损失。

为了解决滚动更新时流量产生损失的情况，推荐在后端服务的Pod中配置preStop Hook，在端点被摘除后仍继续工作一段时间，解决流量中断的情况。

在Pod模板的容器配置中加入：

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        lifecycle:
          # 配置preStop Hook，等待30秒后退出。
          # 需要容器中存在sleep命令。
          preStop:
            exec:
              command:
              - sleep
              - 30
```

