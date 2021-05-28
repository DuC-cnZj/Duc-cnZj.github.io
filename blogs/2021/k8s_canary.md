---
title: 使用 k8s 实现灰度发布
date: '2021-05-12 12:54:14'
sidebar: false
categories:
 - 技术
tags:
 - 灰度发布/金丝雀发布
 - ingress
 - k8s
publish: true
---

对比：ingress controller vs istio

1. ingress canary 特性能实现按百分比发布
2. ingress 简单
3. istio 不仅能实现灰度，蓝绿，还能实现流量转移，故障注入等 https://istio.io/latest/docs/
4. istio 比较重，学习成本很高，而且几年了还是不温不火，部署需要占用不少系统资源
5. istio,ingress 都可以针对请求头做灰度发布，也就是对指定用户做灰度

## ingress controller 实现灰度发布

> [doc](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/#canary)

实现原理：

1. 实现准备 v1 v2 两个版本的镜像

2. 创建 service-v1 service-v2, 使得两个服务分别解析到不同版本的 pod

3. 创建两个 ingress: ingress-v1 ingress-v2, 其中一个增加如下注释

   - ```yaml
     # 开启金丝雀（灰度发布）
     nginx.ingress.kubernetes.io/canary: "true"
     # 权重
     nginx.ingress.kubernetes.io/canary-weight: "90"
     
     # header 包含这个头，值为always时走canary，never不走
     nginx.ingress.kubernetes.io/canary-by-header: "canary"
     # 自定义header匹配走canary的值
     # nginx.ingress.kubernetes.io/canary-by-header-value: "true"
     # cookie中包含 canary: always 走 canary
     nginx.ingress.kubernetes.io/canary-by-cookie: "canary"
     ```

然后通过不断调整比例来实现发布c