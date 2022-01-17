---
title: gitlab runner build 镜像网络问题
date: '2021-06-30 20:01:49'
sidebar: false
categories:
 - 技术
tags:
 - gitlab runner
 - network
 - docker
publish: true
sticky: 20
---


### （没有用！）起初认为这个可以解决（没啥用）
https://docs.gitlab.com/runner/configuration/advanced-configuration.html#helper-image

没有用！
### （没有用！）之后发现然并卵
原因在于 "dind" 镜像本身的 dns, 因为我们公司的 dind 是自己写的 Dockefile，所以问题在这里！！！！

## 有用！！！

因为 MTU 的关系！！！！！
https://gitlab.com/gitlab-org/gitlab-runner/-/issues/3705
https://github.com/projectcalico/calico/issues/2334
https://docs.projectcalico.org/networking/mtu

mtu 修改之后需要重启docker！！！