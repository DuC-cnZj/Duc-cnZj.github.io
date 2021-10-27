---
title:  k8s 笔记
date: '2021-07-05 12:03:24'
sidebar: false
categories:
 - 技术
tags:
 - k8s
publish: true
---


[ReadWriteOnce 原本以为只有一个pod可以读写❌，应该是同一节点上的多个 Pod 可以读取和写入同一个卷, ReadWriteOncePod 才是只有一个pod可以读写](https://kubernetes.io/blog/2021/09/13/read-write-once-pod-access-mode-alpha/#how-is-this-different-than-the-readwriteonce-access-mode)



[迁移 pv, pv和pvc换绑定](https://kubernetes.io/blog/2021/09/13/read-write-once-pod-access-mode-alpha/#migrating-existing-persistentvolumes)

操作步骤

1. 设置pv模式 `kubectl patch pv cat-pictures-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`
2. `kubectl scale --replicas=0 deployment cat-pictures-writer` pod数量改成 0
3. 删除 pvc 或者 helm uninstall
4. 编辑 pv 把关联才 claimRef 字段全删除
5. 临时修改 helm charts 的 pvc 增加 `spec.volumeName`  指向原来的那个 pv
6. `helm install` or `kc apply`
7. 如果原来是 delete 模式就改回去`kubectl patch pv cat-pictures-pv -p '{"spec":{"persistentVolumeReclaimPolicy":"Delete"}}'`