---
title: gitlab runner buil 镜像网络问题
date: '2021-06-30 20:01:49'
sidebar: false
categories:
 - 技术
tags:
 - gitlab runner
 - network
 - docker
publish: true
---

### 起初认为这个可以解决（没啥用）
https://docs.gitlab.com/runner/configuration/advanced-configuration.html#helper-image

### 之后发现然并卵
原因在于 "dind" 镜像本身的 dns, 因为我们公司的 dind 是自己写的 Dockefile，所以问题在这里！！！！