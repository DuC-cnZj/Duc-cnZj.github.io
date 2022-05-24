---
title: externalTrafficPolicy-Local 千万别用ack默认的 nginx ingress
date: '2022-02-25 10:57:44'
sidebar: false
categories:
 - 技术
tags:
 - k8s 坑
publish: true
---

将 nginx ingress 换成 daemonset 模式
因为今天出现了有些pod能访问特定url有些不能访问的问题，最后发现是 service externalTrafficPolicy 的问题

参考
https://segmentfault.com/a/1190000020751999

https://kubernetes.github.io/ingress-nginx/deploy/baremetal/

因此将默认的 deployment 换成 daemonset

操作
ack 添加新ingress参考文档 https://help.aliyun.com/document_detail/151524.html?utm_content=g_1000230851&spm=5176.20966629.toubu.3.f2991ddcpxxvD1#title-eoh-z2m-ru5

