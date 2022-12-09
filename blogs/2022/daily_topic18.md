---
title: k8s 每日一问 - kubelet 日志轮换，以及 resource 里面的 ephemeral-storage 文件大小是否会限制日志，可以看下文档，ephemeral-storage = log+file?
date: '2022-12-09 17:12:41'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---

[local-ephemeral-storage](https://kubernetes.io/zh-cn/docs/concepts/configuration/manage-resources-containers/#local-ephemeral-storage)

[日志](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/logging/#logging-at-the-node-level)

> 都是通过 limits 的 ephemeral-storage 计算的

**kubelet** 日志轮换： 通过kubelet 对这个`/var/log/pods/podNamespace_podName_string(podUID)/*.log` 目录下面的日志按照配置进行rotate 

pod 日志目录 `/var/log/pods/podNamespace_podName_string(podUID)/*.log`

stdout/stderr 日志算在 ephemeral-storage 里面的, **如果直接在宿主机目录里面往pod日志目录写东西, 或者写/var/lib/kubelet/pods/${pod-uid}/etc-hosts ，当超出了ephemeral local storage 设定值都会被驱逐**

**而且驱逐之后的日志目录确实会被删除！**

### ephemeral-storage = log+file?

ephemeral-storage = `logs(/var/logs/pods/xxx/*.log)` + `etc-hosts(/var/lib/kubelet/pods/${pod-uid}/etc-hosts )`+`rootFs(cadvisor 提供的数据)`+`pvc usedBytes(ephemeral类型的)`

代码在这个方法里面

```go
func (p *cadvisorStatsProvider) ListPodStats(_ context.Context) ([]statsapi.PodStats, error) {
```

