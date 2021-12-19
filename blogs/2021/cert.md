---
title: 数字证书原理
date: '2021-12-19 12:28:39'
sidebar: false
categories:
 - 技术
tags:
 - 证书
publish: true
---

>  以下两篇文章都讲的很到位，建议细品

1. [数字证书原理-赵化冰的博客 | Zhaohuabing Blog](https://zhaohuabing.com/post/2020-03-19-pki/)
2. [一文带你彻底厘清 Kubernetes 中的证书工作机制-赵化冰的博客 | Zhaohuabing Blog](https://zhaohuabing.com/post/2020-05-19-k8s-certificate/)

CN 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端证书则为证书申请者的姓名；

简称：O 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端单位证书则为证书申请者所在单位名称；

所在城市 (Locality)| 简称：L 字段 
所在省份 (State/Provice)| 简称：ST 字段 
所在国家 (Country)| 简称：C 字段，只能是国家字母缩写，如中国：CN



## 证书中的 hosts

> [kube-apiserver 证书中的 hosts 受限问题 · Issue #185 · opsnull/follow-me-install-kubernetes-cluster · GitHub](https://github.com/opsnull/follow-me-install-kubernetes-cluster/issues/185)

我简单的把证书分为三类：

- server 证书：server auth；
- client 证书：client auth；
- peer 证书：server auth + client auth。

经过我的实践和理解：client 证书可以不用指定 `hosts` 字段，server 证书和 peer 证书必须要指定 `hosts` 字段，否则客户端访问时会提示错误：

```
Unable to connect to the server: x509: cannot validate certificate for 192.168.1.x because it doesn't contain any IP SANs
```

确实有这个问题，用 go 试了下，hosts 必须包含服务的地址不然就会出问题

但是当我在做 HA master （我使用 Keepalived 新增了一个 VIP，并且使用 HAProxy 代理 https 的 kube-apiserver）时，就会存在一个问题：由于 VIP 并不在 kube-apiserver 的 `hosts` 中，导致所有客户端都没有办法通过 VIP 来访问 kube-apiserver （提示 VIP 不在 SANS 中）。



## golang 自签证书实现 https

[Go HTTPS Demo: golang 生成https 自签名证书，及服务端、客户端双向认证代码](https://toscode.gitee.com/huy_admin/Go-HTTPS-Demo)
