---
title: 阿里云 cert-manager connection refused 的问题
date: '2021-07-17 08:21:20'
sidebar: false
categories:
 - 技术
tags:
 - k8s
 - 证书
 - https
publish: true
---

## 问题描述

cert-manager 使用 http01  时出现

"msg"="propagation check failed" "error"="failed to perform self check 。。。 ”

具体参考

[别人也遇到了](https://blog.crazyphper.com/2019/12/25/slb%e5%8f%8d%e4%bb%a3k8s%e6%97%b6%ef%bc%8c%e4%bd%bf%e7%94%a8cert-manager%e9%81%87%e5%88%b0%e7%9a%84%e6%a3%98%e6%89%8b%e9%97%ae%e9%a2%98/)



## 问题解决方案

https://github.com/pragkent/alidns-webhook