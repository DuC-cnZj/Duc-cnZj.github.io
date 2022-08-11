---
title: k8s 每日一问 - kubeadm reset 都清理了什么
date: '2022-08-11 14:11:28'
sidebar: false
categories:
 - 技术
tags:
 - 每日一问
publish: true
---


## 清理

1. /var/lib/kubelet 目录
2. 移除本地 etcd 数据和节点
3. systemctl stop kubelet 
4. 移除上面所有的容器
5. csi socket

```
/etc/kubernetes/pki
// CRISocketContainerd is the containerd CRI endpoint
CRISocketContainerd = "unix:///var/run/containerd/containerd.sock"
// CRISocketCRIO is the cri-o CRI endpoint
CRISocketCRIO = "unix:///var/run/crio/crio.sock"
// CRISocketDocker is the cri-dockerd CRI endpoint
CRISocketDocker = "unix:///var/run/cri-dockerd.sock"
```

6. Root less 模式启动还会清理 用户和组



## 没有清理

 iptable/ipvs
/etc/cni/net.d