---
title: kubeadm k8s高可用安装
date: '2021-06-25 20:21:03'
sidebar: true
categories:
 - 技术
tags:
 - k8s
publish: true
---

## 架构

1. 3 个 master节点
2. n 个 node 节点
3. 两台 nginx+keepalived机器
4. 两台 为 ingress nginx 准备的机器， haproxy+keepalived, 用来代理集群所有的节点

## 安装过程遇到的问题

1. 初次安装 `--pod-network-cidrs` 没设置，导致 flannel 安装后集群网络不通
2. calico 也有上面的问题, 不过默认网段是 `10.244.0.0/16`
3. setenforce 和 swap 分区未关闭，导致集群起不来或者ingress不通
4. 国内镜像问题（虽然这个早就知道怎么解决，但还是来吐槽下不只是k8s本身需要的镜像，其他的也大多都是需要科学上网才能拉取的）
5. 虚拟机一定要加 `--apiserver-advertise-address` 不然会出现奇奇怪怪的问题，比如pod一直pending，cluster 10.92.0.1 连接不上

## 总结

帮我更深入理解各组件作用，原来只是会用yaml，现在有稍微深入了些。
熟悉了各种方式安装k8s，kubeadm, rancher, aliYun ack。